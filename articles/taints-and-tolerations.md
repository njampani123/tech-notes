---
layout: default
title: Taints and Tolerations, Explained
---

[← Back to index](../index.html)

# Taints and Tolerations, Explained

These control **which pods can be scheduled on which nodes** — the opposite mechanism from node affinity (which pods *want* a node), taints/tolerations let a node *repel* pods unless they explicitly tolerate it.

## The core idea

- **Taint** = applied to a **node**. Says "don't schedule pods here unless they tolerate me."
- **Toleration** = applied to a **pod**. Says "I'm okay running on a node with this taint."

Tolerations don't *attract* a pod to a node — they just allow scheduling. To force a pod onto specific nodes you'd pair this with node affinity/selectors (see below).

## Taint syntax

```
kubectl taint nodes node1 key=value:effect
```

Example:
```
kubectl taint nodes node1 dedicated=gpu-team:NoSchedule
```

## The three effects

| Effect | Behavior |
|---|---|
| `NoSchedule` | New pods without a matching toleration won't be scheduled. Existing running pods are unaffected — this effect never evicts anything. |
| `PreferNoSchedule` | Soft version — scheduler tries to avoid it, but will place pods there if no other option. |
| `NoExecute` | Evicts already-running pods that don't tolerate it, in addition to blocking new ones. |

## Toleration syntax (pod spec)

```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "gpu-team"
  effect: "NoSchedule"
```

- `operator: Equal` — key/value/effect must all match exactly.
- `operator: Exists` — just checks the key exists (value ignored). Useful for wildcard-style tolerations.
- Omitting `key` entirely tolerates **all** taints (used for things like DaemonSets that must run everywhere, e.g. `kube-proxy`, log/metrics agents).

`NoExecute` tolerations also support `tolerationSeconds` — "tolerate this taint, but evict me after N seconds anyway." This is how Kubernetes handles node problems gracefully: every pod gets default tolerations for `node.kubernetes.io/not-ready` and `node.kubernetes.io/unreachable` with a grace period (commonly 300s), so a brief network blip doesn't trigger an immediate mass eviction.

**Important nuance:** taints are evaluated **per-key**. Tolerating one taint on a node buys a pod nothing against a *different* taint added later — e.g., a pod tolerating `dedicated=gpu-team:NoExecute` gets no protection if the node later gets tainted `node.kubernetes.io/disk-pressure:NoExecute` for an unrelated reason. Every taint is its own gate.

## Common real-world uses

1. **Dedicated nodes** — taint GPU or high-memory nodes so only workloads that specifically tolerate/need them land there.
2. **Control-plane isolation** — master nodes are tainted `node-role.kubernetes.io/control-plane:NoSchedule` by default so regular workloads don't get scheduled on them.
3. **Node problem handling** — Kubernetes auto-taints nodes with conditions like `node.kubernetes.io/not-ready`, `node.kubernetes.io/memory-pressure`, `node.kubernetes.io/disk-pressure` (`NoExecute`), which is how pods get evicted from unhealthy nodes automatically.
4. **DaemonSets** — system daemons (CNI, log shippers, monitoring agents) usually carry blanket tolerations so they run on every node, including tainted ones.

## The trap: taints alone don't dedicate a node to your app

Taints only **repel**; they never **attract**. If you taint a node and add a matching toleration to your pod, that pod is *allowed* on the node — but nothing guarantees it actually lands there, and nothing stops other pods with the same toleration (or blanket DaemonSet tolerations) from landing there too.

To truly dedicate a set of nodes to one workload, you need **both halves**:

| Mechanism | Applied to | Does what |
|---|---|---|
| Taint | Node | Repels all pods *except* those with a matching toleration |
| Toleration | Pod | Lets your pod ignore that repulsion |
| `nodeSelector` / `nodeAffinity` | Pod | **Attracts** your pod to that node (positive selection) |

### The standard pattern: label + taint + toleration + affinity

**1. Label the dedicated nodes:**
```
kubectl label nodes node1 workload=gpu-team
```

**2. Taint them so nothing else lands there:**
```
kubectl taint nodes node1 dedicated=gpu-team:NoSchedule
```

**3. On your pod, add both the toleration (get past the taint) and a nodeSelector/affinity (actually go there):**

```yaml
spec:
  nodeSelector:
    workload: gpu-team
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu-team"
    effect: "NoSchedule"
```

Or with `nodeAffinity` for richer matching (multiple values, preferred vs. required):
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: workload
            operator: In
            values: ["gpu-team"]
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu-team"
    effect: "NoSchedule"
```

**Why both are needed:**

- **Taint only** → node is protected from *other* workloads, but your app might not even land there — the scheduler is free to place it anywhere untainted.
- **nodeSelector/affinity only** → your app is *directed* there, but other teams' pods can still pile onto the same nodes (no isolation) unless they're also blocked.
- **Both together** → exclusive, dedicated nodes: only your app goes there, and only your app can go there.

## Edge cases worth knowing

- **Removing a taint never evicts anything.** Removal only ever loosens constraints — it can't retroactively evict a pod that was already running under the old rules.
- **A stray toleration with no matching taint is a complete no-op.** If the taint is removed but the pod spec still carries the toleration, it's just dead config — harmless, but worth cleaning up so it doesn't mislead the next engineer into thinking there's still a dedicated-node setup in place.
- **If a node is still labeled but the taint is gone**, `nodeSelector` becomes the only thing still doing real work — the next pod deployed from that spec will still land on that node (assuming capacity), deterministically, because the label match is unaffected by the taint's removal.
- **Default behavior with zero taints/tolerations anywhere:** the scheduler treats all untainted nodes as fair game. If you only add a toleration without a nodeSelector, don't be surprised when the pod lands on a completely different, unintended node — the scheduler has no reason to prefer the tainted one.

## Mental model

> Taints push pods away. Tolerations just let a pod ignore that push. They don't pull.

If you want the reverse — a pod that *must* land on a specific set of nodes — combine tolerations (to survive the taint) with `nodeAffinity` or `nodeSelector` (to actually target the labeled node).
