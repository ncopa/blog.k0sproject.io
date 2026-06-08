---
title: "k0smotron v2.0.0"
date: 2026-06-04T09:00:00Z
author: "Adrian Pedriza"
tags: ["k0smotron", "kubernetes", "cluster API", "v2.0.0"]
aliases:
    - "/k0smotron-v2.0.0"
cover:
  image: "1.png"
---

k0smotron v2.0.0 is here. This is a big release: a new API version, new storage backends, Windows support, better operator internals, and more. The version bump to `v2.0.0` is not arbitrary, it reflects the promotion of `v1beta2` as the new [resource storage version](https://kubernetes.io/docs/concepts/overview/working-with-objects/storage-version/) across all k0smotron API groups, a change that affects every existing k0smotron user and requires attention before upgrading. On top of that, the release includes a handful of bug fixes and small improvements. Check the [changelog](https://github.com/k0sproject/k0smotron/releases/tag/v2.0.0) for the full list. We will cover the major features in detail in a series of posts coming soon. This post gives you the full picture.

## Windows workers ([#1317](https://github.com/k0sproject/k0smotron/pull/1317))

k0smotron can now bootstrap Windows worker nodes through the CAPI bootstrap provider. This extends the reach of k0smotron-managed clusters to mixed Linux/Windows workloads without requiring manual node setup outside of Cluster API. The new `spec.provisioner` field in `K0sWorkerConfig` is the entry point for this. It consolidates provisioner-specific configuration, including how Windows nodes are bootstrapped. Check the [Windows workers docs](https://docs.k0smotron.io/stable/windows-workers) for a full walkthrough.

## Resource patches ([#1341](https://github.com/k0sproject/k0smotron/pull/1341))

k0smotron generates a set of Kubernetes resources to run each hosted control plane: `StatefulSet`, `Service`, `ConfigMap`, and others. Until now, those resources were mostly opaque. You could configure them through the k0smotron spec, but anything not exposed was out of reach. Resource patches change that. By defining `spec.patches` on a `Cluster` or `K0smotronControlPlane`, you can target any generated resource by kind and component label and apply `strategic`, `merge`, or `json` patches on top. This is particularly useful for tweaking resource requests, adding annotations, or adjusting probe timing without needing a fork. More details in the [component customization docs](https://docs.k0smotron.io/stable/advanced/components/?h=pat#component-customization).

## Embedded NATS storage ([#1380](https://github.com/k0sproject/k0smotron/pull/1380))

k0smotron now supports running an embedded [NATS](https://nats.io) server with JetStream as the control plane storage backend. Each k0s pod runs its own NATS instance, using [kine](https://github.com/k3s-io/kine) to bridge Kubernetes API server operations to JetStream key-value storage. The result is a self-contained control plane with no external etcd or database dependency. This backend is marked **experimental** for now. HA clustering is supported, but the replica count must be set at creation time and cannot be changed. More details in the [NATS storage docs](https://docs.k0smotron.io/stable/nats/?h=nats#embedded-nats-storage).

## Workload cluster connectivity improvements

Two related changes improve how k0smotron connects to the clusters it manages.

**ClusterCache** ([#1408](https://github.com/k0sproject/k0smotron/pull/1408)): k0smotron now uses the CAPI [ClusterCache](https://pkg.go.dev/sigs.k8s.io/cluster-api@v1.11.4/controllers/clustercache) to manage connections to workload clusters. Rather than opening a new connection on every reconciliation, connections and client caches are reused across reconciliation loops. The practical effect is lower resource consumption and more stable behavior in environments with a large number of managed clusters.

**Tunneling client** ([#1433](https://github.com/k0sproject/k0smotron/pull/1433)): when a tunnel to the workload cluster is available through CAPI, k0smotron now uses it preferentially over a direct connection. This improves connectivity in environments where workload clusters are not directly reachable from the management cluster, which is common in air-gapped or private network setups.

## New v1beta2 API version

Promoting `v1beta2` to resource storage version means that all resources are now persisted using the new schema. The old `v1beta1` API is still served via conversion webhooks, but it is officially deprecated and will be removed no earlier than 9 months or 3 minor releases after deprecation, following the [Kubernetes API deprecation policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/). If you are running k0smotron today, now is the time to start planning the migration.

### What changed in v1beta2?

The `v1beta2` API brings consistency and alignment with the broader Kubernetes and Cluster API ecosystem. Changes span all four API groups:

- `k0smotron.io` — `Cluster` and `JoinTokenRequest`
- `controlplane.cluster.x-k8s.io` — `K0sControlPlane` and `K0smotronControlPlane`
- `bootstrap.cluster.x-k8s.io` — `K0sControllerConfig` and `K0sWorkerConfig`
- `infrastructure.cluster.x-k8s.io` — `RemoteMachine` and `RemoteCluster`

There are two main themes behind these changes. The first theme is **status reporting**: `v1beta1` used custom fields like `status.ready` and `status.reconciliationStatus` that did not integrate well with standard Kubernetes tooling. In `v1beta2`, all resources report state using standard **Conditions**, aligned with how Kubernetes itself and the CAPI project work. Legacy fields are preserved under `status.deprecated.v1beta1` during the transition. You can check the [Cluster API proposal about improving status in CAPI resources](https://github.com/kubernetes-sigs/cluster-api/blob/main/docs/proposals/20240916-improve-status-in-CAPI-resources.md) for more information about what motivated these changes.

The second theme is **spec improvements**: the opportunity was taken to clean up and restructure some resource specs, consolidating storage configuration, renaming fields for clarity, and aligning with the CAPI v1beta2 contract.

For the full list of changes per resource, check the [migration guide](https://docs.k0smotron.io/stable/update/migrate-v1beta2-api/#migrate-to-v1beta2-apis).

### Migrating from v1beta1

k0smotron ships a CLI tool that automates the manifest conversion. Download the binary for your platform from the [GitHub releases page](https://github.com/k0sproject/k0smotron/releases) and run it against your existing manifests:

```shell
curl -L https://github.com/k0sproject/k0smotron/releases/latest/download/convert-v1beta1-to-v1beta2-linux-amd64 -o convert
chmod +x convert
./convert my-cluster.yaml > my-cluster-v1beta2.yaml
```

The tool handles all the field renames and restructuring described above, and passes through resources from other API groups unchanged. Always review the output before applying. Keep in mind that `spec.storage.type: nats` and `spec.patches` are new in `v1beta2` with no `v1beta1` equivalent, so they cannot be converted back.

Check the [documentation about the migration tool](https://docs.k0smotron.io/stable/update/migrate-v1beta2-api/#migration-tool) for more details.

