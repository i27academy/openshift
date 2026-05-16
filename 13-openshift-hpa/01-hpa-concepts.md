# OpenShift – Horizontal Pod Autoscaler — Concepts

## What is HPA?

The **Horizontal Pod Autoscaler (HPA)** automatically scales the number of pod replicas in a Deployment up or down based on observed resource usage (CPU, memory, or custom metrics).

Instead of manually adjusting replicas when load increases or decreases, HPA monitors metrics and makes scaling decisions continuously.

```
Low load  → HPA scales down → fewer replicas → less cost
High load → HPA scales up   → more replicas  → handles traffic
```

---

## HPA vs Manual Scaling

| | Manual Scaling | HPA |
|--|---------------|-----|
| Who scales | You (manually) | OpenShift (automatically) |
| When | Whenever you notice load | Continuously, based on metrics |
| Reaction time | Minutes (human) | Seconds (automated loop) |
| Risk | Over/under provisioned | Self-correcting |
| Use case | Predictable, known load | Variable, unpredictable load |

---

## How HPA Works

HPA runs a control loop every **15 seconds** by default:

1. Collects current resource usage from the **Metrics Server**
2. Compares actual usage against the target threshold
3. Calculates the desired number of replicas using the formula:

```
desiredReplicas = currentReplicas × (currentUsage / targetUsage)
```

4. Scales the Deployment up or down accordingly
5. Respects `minReplicas` and `maxReplicas` boundaries

---

## Key HPA Fields

| Field | Meaning |
|-------|---------|
| `minReplicas` | Minimum number of pods — HPA will never scale below this |
| `maxReplicas` | Maximum number of pods — HPA will never scale above this |
| `targetCPUUtilizationPercentage` | Target average CPU usage across all pods as a % of their CPU **request** |

---

## Why Resource Requests Are Required

HPA measures CPU usage as a **percentage of the pod's CPU request**.

```
CPU utilization % = actual CPU usage / CPU request × 100
```

If no CPU request is set on the container, HPA cannot calculate utilization percentage and will show `<unknown>` in the TARGETS column — and will not scale.

**Always set resource requests on any Deployment managed by HPA.**

---

## Metrics Server

HPA relies on the **Metrics Server** to collect real-time CPU and memory usage from pods and nodes.

In OpenShift, the Metrics Server is typically pre-installed. Verify it is running:

```bash
oc get pods -n openshift-monitoring | grep metrics
```

Or check if top commands work:

```bash
oc adm top pods -n i27-hpa-demo
oc adm top nodes
```

If these commands return data, the Metrics Server is available and HPA will function correctly.

---

## Scale-Up and Scale-Down Behaviour

### Scale-Up

HPA scales up quickly when load exceeds the target threshold. New pods are added and begin receiving traffic once they pass readiness checks.

### Scale-Down (Cool-Down Period)

HPA waits before scaling down to avoid thrashing (rapidly scaling up and down). By default, OpenShift waits **5 minutes** after the last scale-up before scaling down. This prevents premature scale-down during brief traffic lulls.

---

## Project Setup

```bash
oc adm new-project i27-hpa-demo
oc project i27-hpa-demo
```

Apply SCC (required for the images used in this lab):

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-hpa-demo
```
