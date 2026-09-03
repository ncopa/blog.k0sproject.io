---
title: "Inside k0s Autopilot: How a Cluster Upgrades Itself"
date: 2026-09-03T09:00:00Z
author: "Natanael Copa"
tags: ["k0s", "kubernetes", "autopilot", "upgrades", "internals"]
draft: false
cover:
  image: "signal.jpg"
  caption: Photo by [Nopparuj Lamaikul](https://unsplash.com/@center999) on [Unsplash](https://unsplash.com)
---

You apply a `Plan` object naming a k0s version and the nodes it should cover, and
some minutes later those nodes are running it. No SSH loop, no external
orchestrator, no node left half-upgraded. That is
[autopilot](https://docs.k0sproject.io/stable/autopilot/), and this post is an
overview of what happens in between.

This article describes k0s
[v1.36.4+k0s.0](https://github.com/k0sproject/k0s/tree/v1.36.4+k0s.0/pkg/autopilot),
and the code links point there.

## The awkward parts of upgrading yourself

Replacing the `k0s` binary on a node and restarting it is easy. Doing it across a
cluster runs into four things that make it more interesting than a loop over
nodes.

The first is that the thing being upgraded is the thing doing the upgrading. A
controller that replaces its own binary has to restart itself, which kills the
process that was tracking how far it had got.

The second is quorum. Three controllers need two of them to keep etcd working, so
taking two down at once loses the majority. Controller updates have to be strictly
sequential, and each one has to be confirmed healthy before the next starts.

The third is workloads. A node carrying pods needs draining first, and draining
asks the eviction API to remove each pod rather than deleting it outright. If an
eviction would take a workload below the replicas its
[PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets)
requires, the API server rejects it. Each drain attempt gets two minutes; if it
does not finish, autopilot tries again, and the node waits there until the drain
succeeds.

The fourth is that you need per-node truth about what happened, and you need the
rollout to stop rather than continue past a node that failed.

## Two state machines, joined by an annotation

Autopilot splits the work in two. A cluster-wide **plan** decides who upgrades
and in what order. A per-node **signal** carries out the upgrade on one node.

```goat
  +---------------------------------------------------------+
  |  PLAN     one leader, cluster wide                      |
  |           "upgrade these nodes to v1.2.3"               |
  +---------------------------------------------------------+
                     |
                     |  writes signalling into annotations
                     |  reads the result back
                     v
  +---------------------------------------------------------+
  |  SIGNAL   one state machine per node                    |
  |           download, swap the binary, restart            |
  +---------------------------------------------------------+
```

The joint between them is the interesting design decision. The plan lives in a
`Plan` object. A node's progress lives in
[annotations](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/signaling/v2/signaling_v2.go#L23-L24)
**on that node's own object**, `ControlNode` for controllers and `Node` for
workers:

```text
k0sproject.io/autopilot-signal-version: v2
k0sproject.io/autopilot-signal-data: {"planId":..., "command":{...},
                                      "status":{"status":"Downloading", ...}}
```

That is what makes surviving your own restart possible. The phase is not held in
memory, it is written on the object. When a controller comes back after replacing
its binary, the new process reads the annotation and picks up from there. (There
is one small piece of in-process state, at the restart phase, and it is the
subject of a later section.)

## The plan side

Only the elected leader runs the plan loop, so exactly one actor ever hands out
work.

```goat
   Plan created
        |
        v
  +---------+  removed APIs still in use?
  | NewPlan | -----------------------------> Warning, and stop
  +---------+
        |
        |  discover targets, all of them SignalPending
        v
  +-----------------+  is there room for another node?
  | SchedulableWait | -------------------------------> +-------------+
  +-----------------+                                  | Schedulable |
        ^                                              +-------------+
        |                                                    |
        |           signal ONE node, mark it SignalSent      |
        +----------------------------------------------------+
        |
        |  every target reached Completed
        v
  +-----------+
  | Completed |
  +-----------+
```

`SchedulableWait` is the gate. It watches the nodes it has already signalled,
and when there is room it flips the plan to `Schedulable`, which releases exactly
one more node and comes straight back. For controllers there is room only once
the previous one has finished, which is how "one controller at a time" is
enforced and why quorum survives. Workers can go several at a time, as many as
the plan's `limits.concurrent` allows, though that defaults to 1. Either way
they only start once every controller is done.

Before releasing a controller, autopilot also
[probes](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/readyprober.go#L31)
`/readyz?verbose` on each controller named in the plan, so a rollout will not
proceed past a controller that is unhealthy for reasons of its own.

[`NewPlan`](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/plans/cmdprovider/k0supdate/newplan.go)
is where the safety checks live, including
[a scan for resources](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/checks/checks.go#L22)
whose APIs are removed in the target version. If your cluster still has objects
the new Kubernetes will not serve, the plan stops at `Warning` before touching
anything. Setting `forceupdate` skips that scan.

Each command type plugs into those states through
[a small interface](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/plans/core/types.go#L61-L74),
so `k0supdate` and `airgapupdate` share the same loop:

```go
type PlanCommandProvider interface {
	CommandID() string
	NewPlan(...)         (apv1beta2.PlanStateType, bool, error)
	Schedulable(...)     (apv1beta2.PlanStateType, bool, error)
	SchedulableWait(...) (apv1beta2.PlanStateType, bool, error)
}
```

## The signal side

What they do on the node differs. A k0s update walks the full chain below. An
airgap update is much shorter: it downloads the image bundle and is done, so it
runs no status, then `Downloading`, then `Completed`.

```goat
  +------------------+  the running version already matches?
  |  no status yet   | ---------------------------------------> Completed
  +------------------+
           |
           v
  +------------------+  download or checksum failed?
  |   Downloading    | ---------------------------------------> FailedDownload,
  +------------------+                                          and the plan stops
           |
           v
  +------------------+
  |    Cordoning     |  drain, on worker and controller+worker nodes only
  +------------------+
           |
           v
  +------------------+
  |  ApplyingUpdate  |  replace the running binary
  +------------------+
           |
           v
  +------------------+
  |     Restart      |  k0s is terminated here
  +------------------+
           |
           |  the new process picks up from here
           v
  +------------------+
  |   UnCordoning    |
  +------------------+
           |
           v
  +------------------+
  |    Completed     |  the plan polls and sees this
  +------------------+
```

The download
[checks a sha256](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/download/downloader.go#L90-L95)
if the plan supplies one, and otherwise skips the check.

`Cordoning` is where a PodDisruptionBudget can hold up the rollout: the plan
cannot advance past a node that will not drain. Only nodes that run pods are
drained, which means workers and controller+worker nodes. A controller-only node
[skips this phase](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/signal/k0s/cordon.go#L284-L295)
and moves straight on.

[`ApplyingUpdate`](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/signal/k0s/apply.go#L138-L150)
has a detail worth pointing out. It hard-links the downloaded binary to a second
name before renaming it over the running one, so the download survives the
rename. If writing the next phase fails, the whole sequence can be replayed
without downloading anything again.

## The phase that outlives its own process

`Restart` is the phase the rest of the design bends around, and it is the only
one handled by two reconcilers instead of one.

The apply phase writes `Restart` and stops there. That write wakes the first of
them, `restart`, which
[terminates k0s](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/signal/k0s/restart_unix.go#L189-L194).
The process dies mid-rollout.

Nothing in autopilot brings it back. The whole restart is that one signal, and
since autopilot runs inside the k0s process, it is signalling itself. Starting
k0s again is the job of the node's service manager, normally the one
[`k0s install`](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/install/service_linux.go#L8-L30)
set up: the systemd unit it writes carries `Restart=always` and `RestartSec=10`,
and the [OpenRC script](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/install/openrc_linux.go#L9)
runs k0s under `supervise-daemon`. The new binary is already in place by then, so
whatever the supervisor starts next is the new version.

When k0s starts again, its watch lists the node object and sees the status still
sitting at `Restart`. That is not a write, it is a fresh start, and the *other*
reconciler,
[`restarted`](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/signal/k0s/restarted_unix.go#L36-L49),
is the one listening for that. So the two are told apart by what woke them:

```goat
  +----------------+  writes Restart   +---------+
  | ApplyingUpdate | ----------------> | restart |  terminates k0s
  +----------------+                   +---------+
                                            |
                                            v
                               +-------------------------+
                               | supervisor restarts k0s |
                               +-------------------------+
                                            |
                                            |  process start
                                            v
                                      +-----------+
                                      | restarted |  moves to UnCordoning
                                      +-----------+
```

A write means the restart has not happened yet, so terminate. A process start
means it already has, so move on. There is also a small in-memory note, set just
before the status is written, as a second guard for the case where both could
otherwise fire.

Either way there is one more condition: the version k0s now reports has to match
what the plan asked for, unless the plan is a forced update. If a node comes back
still reporting the wrong version, neither reconciler advances it. The node stays
at `Restart`, and since nothing times out a signalled node, the plan waits for it
indefinitely.

The same goes for the supervisor. Autopilot never checks that one is there, so a
k0s started by hand in the foreground goes down at this phase and stays down.
Those are the sharp edges of this design: it would rather stall than guess.

## How the phases are wired together

There is no orchestrator walking a node through the phases.
[Each phase](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/signal/k0s/init.go#L119-L174)
is a separate controller-runtime reconciler watching the same object, picked out
by the status it handles:

```goat
  +--------------------------------------------------------+
  |  the k0s update chain                                  |
  |                                                        |
  |  status: no status yet   -> signal                     |
  |  status: Downloading     -> download                   |
  |  status: Cordoning       -> cordon      (leader side)  |
  |  status: ApplyingUpdate  -> apply                      |
  |  status: UnCordoning     -> uncordon    (leader side)  |
  |                                                        |
  |  status: Restart, on a write       -> restart          |
  |  status: Restart, on process start -> restarted        |
  +--------------------------------------------------------+
```

Most of the chain drives itself: a phase writes the next status, and that write
wakes the phase that handles it. Those five wake on a fresh process start too,
which is how a node that was interrupted mid-phase resumes on its own.

`Restart` is the exception, and that is the whole point of the previous section.
Because it hands over by restarting the process rather than by writing anything,
its two reconcilers have to tell a write apart from a process start, and they are
the only pair in the chain that does.

Airgap updates use their own, shorter set of filters, so this table is the k0s
update chain rather than a picture of autopilot in general.

## Where each piece runs

```goat
  +------------------------+  +----------------------+  +----------------+
  | CONTROLLER, leader     |  | CONTROLLER, follower |  | WORKER         |
  |                        |  |                      |  |                |
  | plan loop              |  |                      |  |                |
  | update checker         |  | update checker       |  |                |
  | its own phases         |  | its own phases       |  | its own phases |
  |                        |  |                      |  |                |
  | cordons and uncordons  |  |                      |  |                |
  | the other nodes        |  |                      |  |                |
  +------------------------+  +----------------------+  +----------------+
```

Most phases are filtered by hostname, so a node downloads, swaps and restarts
only itself. Draining is the exception. In 1.36.4 cordoning and uncordoning are
done centrally: on the elected leader those two reconcilers
[drop the hostname filter](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/signal/k0s/init.go#L134-L137)
and act on every node in the cluster. The code carries a note to move this to
leaders only in a later release.

The plan loop runs on the leader alone. The update checker, which polls for new
releases, runs on every controller.

Controller+worker nodes are the awkward case. Such a node runs the controller
component but is also a worker, so the plan writes airgap signalling to its
`Node` object while only the `ControlNode` chain would normally be listening.
Autopilot handles it by
[registering the airgap chain a second time](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/signal/init.go#L35-L45)
against the `Node` object when `--enable-worker` is set.

## The window in the middle

Every phase has the same shape, and it is the part that needs the most care:

```goat
  +-----------+     +--------------+     +-----------------------+
  | read the  | --> | do the work  | --> | write the next status |
  | node      |     | seconds to   |     |                       |
  +-----------+     | minutes      |     +-----------------------+
       ^            +--------------+                 |
       |                                             |
       +---- the node can change in here ------------+
```

The work takes time. A drain can take minutes, and plenty can happen while it
runs. Something else writes to the same object. A reconcile is triggered by an
event that was already stale when it arrived. A retry re-runs a phase long after
the signal moved on. If a phase acted on what it read at the start and wrote the
result regardless, a node could be pushed back into a phase it had already left.

Two things keep that from happening in 1.36.4. The first came in
[#6982](https://github.com/k0sproject/k0s/pull/6982): every phase
[checks up front](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/signal/common/download.go#L64-L67)
that the status really is the one it handles, and returns immediately if it is
not.

```go
if signalData.Status != nil && signalData.Status.Status != Downloading {
	logger.Debug("Ignoring signal status ", signalData.Status.Status)
	return cr.Result{}, nil
}
```

That is what makes a late or duplicated event harmless: the download reconciler
woken up for a node that has already reached `Cordoning` simply declines to act.

The second is that the write is an ordinary Kubernetes update, which carries the
resource version of the object it was based on. If the node changed since it was
read, the API server rejects the write and the reconcile is queued again, rather
than quietly overwriting whatever landed in the meantime. The retry redoes the
phase's work from the top, and this time the check above sees the current state.

Together they stop a node from being moved backwards. The
[`ap-controllerworker` integration test](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/inttest/ap-controllerworker/controllerworker_test.go)
keeps them honest: it
[writes unrelated annotations](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/inttest/ap-controllerworker/controllerworker_test.go#L190-L208)
to the node objects for the whole duration of an upgrade, and
[fails if any node](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/inttest/ap-controllerworker/controllerworker_test.go#L262-L269)
is ever seen going backwards.

Moving forwards is a weaker promise. Where autopilot cannot confirm that a phase
really happened, it stops instead of advancing, which is what the restart phase
does when its version check fails. Stopping is the safer of the two: a node stuck
mid-plan is easy to spot, a node wrongly marked as done is not.

## Further reading

The code follows the same shape as this post.
[`plans/core/types.go`](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/plans/core/types.go)
has the contract,
[`plans/cmdprovider/k0supdate/`](https://github.com/k0sproject/k0s/blob/v1.36.4+k0s.0/pkg/autopilot/controller/plans/cmdprovider/k0supdate)
has the plan loop split across `newplan.go`, `schedulable.go` and
`schedulablewait.go`, and
[`signal/k0s/`](https://github.com/k0sproject/k0s/tree/v1.36.4+k0s.0/pkg/autopilot/controller/signal/k0s)
has one file per phase, wired together in `init.go`.

- [Autopilot documentation](https://docs.k0sproject.io/stable/autopilot/)
- [Multi-command plans](https://docs.k0sproject.io/stable/autopilot-multicommand/)
- [Autopilot source](https://github.com/k0sproject/k0s/tree/v1.36.4+k0s.0/pkg/autopilot)
