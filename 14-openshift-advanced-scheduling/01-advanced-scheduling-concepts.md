# OpenShift – Advanced Scheduling — Concepts

## What is Scheduling?

When a pod is created, the OpenShift scheduler decides **which node** to run it on. The default scheduler looks at:

- Node resource availability (CPU, memory)
- Pod resource requests
- Basic constraints like namespace limits

For most workloads, the default scheduler works fine. But real-world production clusters have more complex requirements — dedicated nodes for specific workloads, spreading replicas for high availability, co-locating services that communicate heavily, or isolating sensitive workloads from general traffic.

Advanced scheduling gives you precise control over where pods land.

---

## Why Advanced Scheduling?

| Scenario | Need |
|----------|------|
| App requires SSD-backed nodes | Schedule only on nodes with SSD |
| Database pods must not share a node | Spread across different nodes |
| Frontend must run close to backend | Co-locate on same node or zone |
| Dedicated nodes for GPU workloads | Prevent non-GPU pods from using those nodes |
| Infra workloads on infra nodes only | Isolate from application workloads |

---

## Scheduling Mechanisms Overview

### NodeSelector

The simplest form. A pod declares which node labels it requires. The scheduler only places the pod on nodes that have all the specified labels.

- Hard rule only — no soft preference
- Best for simple, binary node selection

---

### NodeAffinity

An advanced version of NodeSelector with more expressive rules:

- **Required (hard)** — pod must be placed on a matching node or it stays Pending
- **Preferred (soft)** — scheduler tries to match but falls back to any node if no match found
- Supports operators: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`

---

### PodAffinity and PodAntiAffinity

Instead of targeting nodes by their labels, these target nodes based on **which other pods are already running on them**.

- **PodAffinity** — schedule this pod on a node where specific pods are already running (co-location)
- **PodAntiAffinity** — schedule this pod on a node where specific pods are NOT running (spreading)

Both support hard (required) and soft (preferred) variants.

---

### Taints and Tolerations

Works in the opposite direction from affinity:

- **Taint** — applied to a node, repels pods from being scheduled there
- **Toleration** — applied to a pod, allows it to be scheduled on a tainted node

Taints protect nodes from receiving unwanted pods. Only pods that explicitly tolerate a taint can land on that node.

---

## When to Use Which

| Mechanism | Direction | Hard/Soft | Use Case |
|-----------|-----------|-----------|----------|
| NodeSelector | Pod → Node | Hard only | Simple node targeting by label |
| NodeAffinity | Pod → Node | Both | Complex node targeting with operators |
| PodAffinity | Pod → Pod (same node) | Both | Co-locate related pods |
| PodAntiAffinity | Pod → Pod (different node) | Both | Spread replicas for HA |
| Taints | Node → repels Pods | N/A | Dedicate nodes, isolate workloads |
| Tolerations | Pod → accepts Taint | N/A | Allow pod onto tainted node |

---

## Project Setup

```bash
oc adm new-project i27-scheduling-demo
oc project i27-scheduling-demo
```

Apply SCC:

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-scheduling-demo
```

Check available nodes:

```bash
oc get nodes
```

Check existing node labels:

```bash
oc get nodes --show-labels
```
