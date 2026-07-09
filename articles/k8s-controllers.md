---
layout: default
title: Kubernetes Controllers, Explained
---

[← Back to index](../index.html)

# Kubernetes Controllers, Explained

## The core pattern first

Every controller in Kubernetes follows the same loop, regardless of what it manages:

```
watch desired state (spec) → compare to actual state (status) → take action to converge them → repeat forever
```

This is the **reconciliation loop** (also called a "control loop"). It's level-triggered, not edge-triggered — controllers don't react to individual events so much as continuously re-check "is reality what the spec says it should be?" That's why Kubernetes is self-healing: kill a pod manually, and within seconds a controller notices the actual count no longer matches the desired count and creates a replacement.

Most built-in controllers run as goroutines inside a single binary: **`kube-controller-manager`**. It's not one controller — it's dozens bundled into one process for operational simplicity.

## Built-in controllers, grouped by what they manage

### Workload controllers (the ones you interact with daily)

| Controller | Job |
|---|---|
| **ReplicaSet controller** | Ensures N identical pod replicas exist. The actual thing that creates/deletes pods for scaling — Deployments don't touch pods directly, they manage ReplicaSets, which manage pods. |
| **Deployment controller** | Manages ReplicaSets to do rolling updates/rollbacks. Creates a new ReplicaSet on spec change, scales it up while scaling the old one down. |
| **StatefulSet controller** | Like ReplicaSet but preserves pod identity — stable network names (`pod-0`, `pod-1`...), stable storage (PVC per pod), ordered creation/deletion. |
| **DaemonSet controller** | Ensures exactly one pod per node (or per matching node subset). Watches node additions/removals and schedules accordingly — this is how log agents/CNI pods get everywhere automatically. |
| **Job controller** | Runs pods to completion, tracks success count, retries on failure up to `backoffLimit`. |
| **CronJob controller** | Creates Jobs on a schedule. Handles `concurrencyPolicy` (Allow/Forbid/Replace) when a run overlaps the previous one. |

### Cluster infrastructure controllers

| Controller | Job |
|---|---|
| **Node controller** | Watches node heartbeats; marks nodes `NotReady` after missing heartbeats, applies `NoExecute` taints (`node.kubernetes.io/not-ready`) automatically, evicts pods after the grace period. |
| **Endpoint(Slice) controller** | Watches Services + matching pod readiness, keeps the Endpoints/EndpointSlice objects up to date. |
| **Namespace controller** | Handles namespace deletion — cascades deletion of everything inside it. |
| **ServiceAccount controller** | Ensures every namespace has a `default` ServiceAccount and (older clusters) manages its token Secret. |
| **PersistentVolume controller** (`PersistentVolumeBinder`) | Binds PVCs to matching PVs, handles dynamic provisioning by calling out to the StorageClass's provisioner. |
| **AttachDetach controller** | Attaches/detaches volumes to nodes as pods using them get scheduled/removed. |
| **GarbageCollector controller** | Cleans up dependent objects when an owner is deleted (owner references) — e.g., deleting a ReplicaSet cleans up its pods. |
| **TTL controller** | Deletes finished Jobs/Pods after a configured TTL (`ttlSecondsAfterFinished`). |
| **ResourceQuota controller** | Tracks resource usage per namespace against quota limits, rejects requests that would exceed it. |
| **CertificateSigningRequest controller** | Handles kubelet cert rotation and CSR approval workflows. |

### Autoscaling

- **HPA controller** — technically a separate control loop, not bundled into `kube-controller-manager`. Polls metrics (via metrics-server or custom/external metrics APIs) every ~15s and adjusts `replicas` on the target Deployment/ReplicaSet.

### Cloud-specific (moved to `cloud-controller-manager` on managed clusters like EKS/GKE)

| Controller | Job |
|---|---|
| **Route controller** | Sets up cloud provider network routes between nodes. |
| **Service controller** | Provisions cloud load balancers for `type: LoadBalancer` Services. |
| **Node controller (cloud portion)** | Initializes node metadata from the cloud provider (zone, instance type), removes nodes from Kubernetes when the underlying cloud instance is terminated. |

## Beyond built-in: custom controllers / Operators

Anything with a CRD (Custom Resource Definition) typically ships its own controller reconciling that custom resource — this is the **Operator pattern**. A good example: **ArgoCD's application controller** is exactly this — it reconciles `Application` CRDs against Git + cluster state, using the identical watch → diff → converge loop as every controller above. Same pattern, different domain.

## Mental model to walk away with

> There isn't "one controller" — there's a shared *pattern* (reconcile spec vs status) implemented dozens of times for different resource types, bundled together operationally into `kube-controller-manager` for the built-ins, and infinitely extensible via CRDs + custom controllers for everything else (Operators, ArgoCD, cert-manager, etc.).
