# OpenShift – LimitRange and ResourceQuota

## Overview

Individual pod resource specs protect a single workload. But in a shared cluster with multiple teams and namespaces, you also need controls at the **namespace level**:

- **LimitRange** — sets default, minimum, and maximum resource values per container or pod within a namespace
- **ResourceQuota** — caps the **total** resources that can be consumed across all pods in a namespace

---

## LimitRange

### What it Does

A LimitRange enforces constraints at the container level within a namespace:

- Sets **default requests and limits** applied automatically when a pod does not specify them
- Sets **minimum and maximum** allowed values — pods that exceed the max or fall below the min are rejected

### Why it Matters

Without a LimitRange, a developer can deploy a pod with no resource spec at all — it gets the `BestEffort` QoS class and can consume unlimited resources, starving other workloads.

### LimitRange YAML

**`limitrange.yaml`**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: i27-limitrange
  namespace: i27-resources-demo
spec:
  limits:
  - type: Container
    default:              # applied when container has no limits set
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:       # applied when container has no requests set
      cpu: "100m"
      memory: "128Mi"
    max:                  # container cannot exceed these values
      cpu: "500m"
      memory: "512Mi"
    min:                  # container must request at least these values
      cpu: "50m"
      memory: "64Mi"
```

Apply:

```bash
oc apply -f limitrange.yaml
```

Verify:

```bash
oc get limitrange -n i27-resources-demo
oc describe limitrange i27-limitrange -n i27-resources-demo
```

### Test Default Injection

Deploy a pod with no resource spec:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-resources-demo
  namespace: i27-resources-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: no-resources
  template:
    metadata:
      labels:
        app: no-resources
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]
```

```bash
oc apply -f no-resources-demo.yaml
oc describe pod <pod-name> -n i27-resources-demo
```

The LimitRange defaults are automatically applied to the container — no resource spec needed in the pod YAML.

---

## ResourceQuota

### What it Does

A ResourceQuota sets a hard cap on the **total** resources that can be used across all pods in a namespace:

- If a new pod would push usage over the quota, it stays in `Pending`
- Enforces fair resource sharing between teams and namespaces

### ResourceQuota YAML

**`resourcequota.yaml`**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: i27-resourcequota
  namespace: i27-resources-demo
spec:
  hard:
    requests.cpu: "500m"       # total CPU requests across all pods cannot exceed 500m
    requests.memory: "512Mi"   # total memory requests across all pods cannot exceed 512Mi
    limits.cpu: "1"            # total CPU limits across all pods cannot exceed 1 core
    limits.memory: "1Gi"       # total memory limits across all pods cannot exceed 1Gi
    pods: "5"                  # maximum 5 pods in this namespace
```

Apply:

```bash
oc apply -f resourcequota.yaml
```

Verify:

```bash
oc get resourcequota -n i27-resources-demo
oc describe resourcequota i27-resourcequota -n i27-resources-demo
```

Expected output from describe:

```
Name:            i27-resourcequota
Namespace:       i27-resources-demo
Resource         Used    Hard
--------         ----    ----
limits.cpu       100m    1
limits.memory    200Mi   1Gi
pods             1       5
requests.cpu     50m     500m
requests.memory  100Mi   512Mi
```

---

## Create Project with Quota Using `oc adm`

OpenShift allows creating a project with quota applied directly via `oc adm`:

```bash
oc adm new-project i27-resources-demo \
  --description="Resources demo project"
```

Then apply quota and limitrange separately:

```bash
oc apply -f limitrange.yaml
oc apply -f resourcequota.yaml
```

---

## What Happens When Quota is Exceeded

Scale the deployment to exceed the pod quota:

```bash
oc scale deployment resources-demo --replicas=10 -n i27-resources-demo
```

Check pods:

```bash
oc get pods -n i27-resources-demo
```

Only 5 pods will be created (the quota limit). The ReplicaSet will show an error:

```bash
oc describe replicaset <rs-name> -n i27-resources-demo
```

```
Warning  FailedCreate  Error creating: pods "resources-demo-xxx" is forbidden:
exceeded quota: i27-resourcequota, requested: pods=1, used: pods=5, limited: pods=5
```

---

## LimitRange vs ResourceQuota

| | LimitRange | ResourceQuota |
|--|-----------|--------------|
| Scope | Per container / pod | Entire namespace |
| Controls | Default, min, max per container | Total usage across all pods |
| Rejects when | Pod exceeds max or falls below min | New pod would push namespace over total |
| Use case | Prevent single pod runaway | Fair sharing between teams |
