# OpenShift – NodeSelector

## What is NodeSelector?

NodeSelector is the simplest way to constrain which nodes a pod can be scheduled on. You add a label to specific nodes and then reference that label in the pod spec. The scheduler only places the pod on nodes that have the matching label.

It is a **hard rule** — if no node matches, the pod stays in `Pending`.

---

## Prerequisites

```bash
oc project i27-scheduling-demo
```

---

## Step 1 — Check Available Nodes

```bash
oc get nodes
```

Pick a worker node to label. Replace `<node-name>` with an actual node name from your cluster throughout this lab.

---

## Step 2 — Label a Node

Add a custom label to a specific node:

```bash
oc label node <node-name> disktype=ssd
```

Verify the label was applied:

```bash
oc get nodes --show-labels | grep disktype
```

Or describe the node:

```bash
oc describe node <node-name> | grep disktype
```

---

## Step 3 — Deploy with NodeSelector

**`nodeselector-demo.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeselector-demo
  namespace: i27-scheduling-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodeselector-demo
  template:
    metadata:
      labels:
        app: nodeselector-demo
    spec:
      nodeSelector:
        disktype: ssd       # only schedule on nodes with this label
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
oc apply -f nodeselector-demo.yaml
```

---

## Step 4 — Verify Pods Land on the Correct Node

```bash
oc get pods -o wide -n i27-scheduling-demo
```

Expected output — all pods scheduled on the labeled node:

```
NAME                               READY   STATUS    NODE
nodeselector-demo-xxxxxxx-aaaaa    1/1     Running   <node-name>
nodeselector-demo-xxxxxxx-bbbbb    1/1     Running   <node-name>
nodeselector-demo-xxxxxxx-ccccc    1/1     Running   <node-name>
```

All 3 replicas land on the node labeled `disktype=ssd`.

---

## Step 5 — Simulate No Match (Pending)

Add a nodeSelector that no node satisfies:

```bash
oc patch deployment nodeselector-demo -n i27-scheduling-demo \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"disktype":"nvme"}}}}}'
```

Check pods:

```bash
oc get pods -n i27-scheduling-demo
```

New pods will stay in `Pending`:

```
NAME                               READY   STATUS    NODE
nodeselector-demo-yyyyyyy-aaaaa    0/1     Pending   <none>
```

Describe a pending pod to see the scheduler reason:

```bash
oc describe pod <pending-pod-name> -n i27-scheduling-demo
```

Look for:

```
Events:
  Warning  FailedScheduling  0/3 nodes are available:
  3 node(s) didn't match Pod's node affinity/selector.
```

---

## Step 6 — Restore and Clean Up Label

Restore the correct label:

```bash
oc patch deployment nodeselector-demo -n i27-scheduling-demo \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"disktype":"ssd"}}}}}'
```

Remove the node label when done:

```bash
oc label node <node-name> disktype-
```

The `-` at the end of the label key removes it.

---

## Key Points

- NodeSelector only supports **exact label match** — no operators or expressions
- It is a **hard constraint** — no fallback if no node matches
- For more flexible rules (soft preferences, operators), use NodeAffinity instead
- Multiple labels in `nodeSelector` act as **AND** — all must match
