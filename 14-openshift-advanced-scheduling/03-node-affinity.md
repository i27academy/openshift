# OpenShift – Node Affinity

## What is NodeAffinity?

NodeAffinity is the advanced version of NodeSelector. It uses label expressions instead of exact matches, supports multiple operators, and allows both **hard** (required) and **soft** (preferred) rules.

---

## Hard vs Soft Rules

| Type | Field | Behaviour |
|------|-------|-----------|
| Hard | `requiredDuringSchedulingIgnoredDuringExecution` | Pod must land on a matching node or stays Pending |
| Soft | `preferredDuringSchedulingIgnoredDuringExecution` | Scheduler tries to match but places pod anywhere if no match |

`IgnoredDuringExecution` means if a node's labels change after the pod is scheduled, the pod is **not** evicted. It only affects scheduling decisions, not running pods.

---

## Supported Operators

| Operator | Meaning |
|----------|---------|
| `In` | Node label value is in the specified list |
| `NotIn` | Node label value is not in the specified list |
| `Exists` | Node has the label key (value doesn't matter) |
| `DoesNotExist` | Node does not have the label key |
| `Gt` | Node label value is greater than the specified value |
| `Lt` | Node label value is less than the specified value |

---

## Prerequisites

```bash
oc project i27-scheduling-demo
```

Label nodes for this lab:

```bash
oc label node <node-name-1> zone=us-east-1a
oc label node <node-name-2> zone=us-east-1b
```

Verify:

```bash
oc get nodes --show-labels | grep zone
```

---

## Example 1 — Hard Rule (Required)

Pod **must** be scheduled on a node with `zone` label set to either `us-east-1a` or `us-east-1b`. Any other node is rejected.

**`node-affinity-required.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-required-demo
  namespace: i27-scheduling-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: affinity-required
  template:
    metadata:
      labels:
        app: affinity-required
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: zone
                operator: In
                values:
                - us-east-1a
                - us-east-1b
      containers:
      - name: nginx
        image: devopswithcloudhub/nginx:blue
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
```

Apply:

```bash
oc apply -f node-affinity-required.yaml
```

Verify pods land only on labeled nodes:

```bash
oc get pods -o wide -n i27-scheduling-demo
```

---

## Example 2 — Soft Rule (Preferred)

Scheduler **tries** to place pods on nodes with `zone=us-east-1a` but will use any node if no match is found. The `weight` (1–100) determines how strongly this preference is applied when multiple preferences exist.

**`node-affinity-preferred.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-preferred-demo
  namespace: i27-scheduling-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: affinity-preferred
  template:
    metadata:
      labels:
        app: affinity-preferred
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80               # high preference for this zone
            preference:
              matchExpressions:
              - key: zone
                operator: In
                values:
                - us-east-1a
          - weight: 20               # lower preference for this zone
            preference:
              matchExpressions:
              - key: zone
                operator: In
                values:
                - us-east-1b
      containers:
      - name: nginx
        image: devopswithcloudhub/nginx:blue
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
```

Apply:

```bash
oc apply -f node-affinity-preferred.yaml
```

Pods will prefer `us-east-1a` nodes. If none are available, `us-east-1b` is tried. If neither is available, pods are scheduled on any node — no Pending.

---

## Example 3 — Hard + Soft Combined

Use required to enforce a region constraint and preferred to express a zone preference within that region:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: region                # must be in us-east
          operator: In
          values:
          - us-east
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 70
      preference:
        matchExpressions:
        - key: zone                  # prefer us-east-1a within that region
          operator: In
          values:
          - us-east-1a
```

---

## Verify and Inspect

```bash
oc get pods -o wide -n i27-scheduling-demo
oc describe pod <pod-name> -n i27-scheduling-demo
```

In the describe output, the `Node-Selectors` and `Tolerations` sections show the applied constraints. Check the `Events` section for any scheduling failures.

---

## Clean Up Node Labels

```bash
oc label node <node-name-1> zone-
oc label node <node-name-2> zone-
```

---

## NodeSelector vs NodeAffinity

| | NodeSelector | NodeAffinity |
|--|-------------|-------------|
| Rule type | Hard only | Hard and soft |
| Operators | Exact match only | In, NotIn, Exists, Gt, Lt, etc. |
| Multiple values | No | Yes |
| Soft preference | No | Yes (with weight) |
| Complexity | Simple | More expressive |
