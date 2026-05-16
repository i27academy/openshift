# OpenShift – Taints and Tolerations

## What is a Taint?

A taint is applied to a **node** and tells the scheduler to **repel pods** from being placed there — unless the pod explicitly tolerates the taint.

Taints protect dedicated nodes from receiving general workloads. Only pods that declare a matching toleration can be scheduled on a tainted node.

---

## What is a Toleration?

A toleration is applied to a **pod** and allows it to be scheduled on a node that has a matching taint. A toleration does not force the pod onto the tainted node — it just permits it.

```
Node has Taint → repels all pods
Pod has Toleration → permitted on that node
```

---

## Taint Effects

| Effect | Behaviour |
|--------|-----------|
| `NoSchedule` | New pods without a matching toleration will not be scheduled on this node. Existing pods are not affected. |
| `PreferNoSchedule` | Scheduler tries to avoid placing pods here but will if no other option exists. |
| `NoExecute` | New pods without toleration are not scheduled. Existing pods without toleration are evicted. |

---

## Prerequisites

```bash
oc project i27-scheduling-demo
```

---

## Step 1 — Add a Taint to a Node

Mark a node as dedicated for infra workloads:

```bash
oc adm taint node <node-name> dedicated=infra:NoSchedule
```

Taint format: `key=value:Effect`

Verify the taint:

```bash
oc describe node <node-name> | grep Taints
```

Expected:

```
Taints: dedicated=infra:NoSchedule
```

---

## Step 2 — Deploy Without Toleration (Repelled)

Deploy a pod with no toleration — it should not land on the tainted node:

**`no-toleration-demo.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-toleration-demo
  namespace: i27-scheduling-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: no-toleration
  template:
    metadata:
      labels:
        app: no-toleration
    spec:
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

```bash
oc apply -f no-toleration-demo.yaml
```

Check which nodes pods land on:

```bash
oc get pods -o wide -n i27-scheduling-demo
```

No pods should be on the tainted node. If your cluster has only one worker node and it is tainted, pods will stay `Pending`.

---

## Step 3 — Deploy With Toleration (Permitted)

Add a toleration matching the taint. The pod is now permitted to land on the tainted node.

**`toleration-demo.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: toleration-demo
  namespace: i27-scheduling-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: toleration-demo
  template:
    metadata:
      labels:
        app: toleration-demo
    spec:
      tolerations:
      - key: "dedicated"         # must match taint key
        operator: "Equal"        # Equal checks key=value, Exists checks key only
        value: "infra"           # must match taint value
        effect: "NoSchedule"     # must match taint effect
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
oc apply -f toleration-demo.yaml
```

Verify pods with the toleration can now land on the tainted node:

```bash
oc get pods -o wide -n i27-scheduling-demo
```

---

## Toleration Operators

| Operator | Behaviour |
|----------|-----------|
| `Equal` | Key, value, and effect must all match the taint |
| `Exists` | Only key and effect must match — value is ignored |

Using `Exists` — tolerates any taint with that key regardless of value:

```yaml
tolerations:
- key: "dedicated"
  operator: "Exists"
  effect: "NoSchedule"
```

Tolerate all taints on a node (use carefully):

```yaml
tolerations:
- operator: "Exists"
```

---

## NoExecute — Evicting Running Pods

`NoExecute` is stronger than `NoSchedule` — it also evicts already-running pods that do not have a matching toleration.

Add a `NoExecute` taint:

```bash
oc adm taint node <node-name> maintenance=true:NoExecute
```

Any pods running on that node without a `maintenance=true:NoExecute` toleration will be evicted immediately.

To tolerate `NoExecute` with a grace period (pod stays for 60 seconds before eviction):

```yaml
tolerations:
- key: "maintenance"
  operator: "Equal"
  value: "true"
  effect: "NoExecute"
  tolerationSeconds: 60
```

---

## Taint + NodeAffinity Together (Real-World Pattern)

Taints alone permit a pod onto a node but do not force it there. To exclusively place workloads on dedicated nodes, combine taints with NodeAffinity:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: dedicated
            operator: In
            values:
            - infra                  # NodeAffinity forces pod to infra node
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "infra"
    effect: "NoSchedule"             # Toleration permits pod on tainted node
```

This ensures:
- The pod goes only to the infra node (NodeAffinity)
- The pod is permitted on the tainted infra node (Toleration)

---

## Remove a Taint

Remove a taint by adding `-` at the end:

```bash
oc adm taint node <node-name> dedicated=infra:NoSchedule-
oc adm taint node <node-name> maintenance=true:NoExecute-
```

Verify:

```bash
oc describe node <node-name> | grep Taints
```

Output should show `<none>`.

---

## Taints vs Affinity

| | Taints and Tolerations | Node/Pod Affinity |
|--|----------------------|------------------|
| Applied to | Node (taint) and Pod (toleration) | Pod only |
| Direction | Node repels pods | Pod targets nodes |
| Default behaviour | Repel all unless tolerated | Allow all unless constrained |
| Use case | Dedicated and isolated nodes | Targeted placement |
