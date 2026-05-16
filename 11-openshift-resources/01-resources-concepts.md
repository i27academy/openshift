# OpenShift – Resource Requests and Limits — Concepts

## Why Resource Management?

In a shared cluster, multiple pods run on the same nodes. Without resource controls:

- One pod can consume all CPU on a node, starving other pods
- One pod can consume all memory, causing the node to evict other pods
- The scheduler has no basis for deciding which node can handle a new pod

Requests and limits give OpenShift the information it needs to schedule pods correctly and protect the cluster from runaway workloads.

---

## Requests

A **request** is the amount of CPU or memory the container is **guaranteed** to get.

- The scheduler uses requests to decide which node to place the pod on
- A node must have at least the requested amount of free resources to accept the pod
- The container is always guaranteed this amount, even under heavy cluster load

```yaml
resources:
  requests:
    memory: "100Mi"
    cpu: "50m"
```

---

## Limits

A **limit** is the maximum amount of CPU or memory the container is **allowed** to use.

- The container can use up to this amount but never more
- What happens when a limit is exceeded depends on the resource type:

| Resource | Exceeds Limit | Result |
|----------|--------------|--------|
| CPU | Throttled | Container slowed down, not killed |
| Memory | OOMKilled | Container killed immediately (exit code 137) |

```yaml
resources:
  limits:
    memory: "200Mi"
    cpu: "100m"
```

---

## Requests vs Limits

| | Requests | Limits |
|--|----------|--------|
| Purpose | Scheduling guarantee | Maximum allowed usage |
| Used by | Scheduler (node placement) | Kubelet (enforcement) |
| CPU exceeded | N/A | Throttled |
| Memory exceeded | N/A | OOMKilled (exit 137) |
| Must be set? | Recommended | Recommended |

---

## CPU Units

CPU is measured in **cores** or **millicores**.

| Value | Meaning |
|-------|---------|
| `1` | 1 full CPU core |
| `0.5` | Half a CPU core |
| `100m` | 100 millicores = 0.1 core |
| `50m` | 50 millicores = 0.05 core |

Millicores (`m`) are the standard unit in Kubernetes/OpenShift resource specs. `1000m = 1 core`.

---

## Memory Units

Memory is measured in **bytes** with standard suffixes.

| Value | Meaning |
|-------|---------|
| `128Mi` | 128 mebibytes (1 Mi = 1024 x 1024 bytes) |
| `1Gi` | 1 gibibyte |
| `512M` | 512 megabytes (decimal, slightly different from Mi) |

Always use `Mi` and `Gi` in Kubernetes/OpenShift specs — they are unambiguous.

---

## QoS Classes

OpenShift assigns a **Quality of Service (QoS)** class to each pod based on how requests and limits are set. This determines eviction priority when a node runs out of resources.

| QoS Class | Condition | Eviction Priority |
|-----------|-----------|------------------|
| `Guaranteed` | Requests == Limits for all containers | Last to be evicted |
| `Burstable` | Requests < Limits (or only one is set) | Middle priority |
| `BestEffort` | No requests or limits set at all | First to be evicted |

```bash
# Check QoS class of a pod
oc get pod <pod-name> -o jsonpath='{.status.qosClass}'
```

---

## How the Scheduler Uses Requests

When a pod is created, the scheduler:

1. Looks at each node's **allocatable** resources
2. Subtracts all existing pod **requests** on that node
3. Checks if the remaining capacity can fit the new pod's requests
4. If yes, schedules the pod there

The scheduler does **not** look at actual current usage — only declared requests. This is why setting accurate requests matters.

---

## Project Setup

```bash
oc adm new-project i27-resources-demo
oc project i27-resources-demo
```

Apply SCC (required for the stress image):

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-resources-demo
```
