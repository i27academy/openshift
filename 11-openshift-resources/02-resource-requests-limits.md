# OpenShift – Resource Requests and Limits Lab

## What We Are Doing

Deploy a container using the `polinux/stress` image with explicit CPU and memory requests and limits. The stress tool actively consumes memory and CPU so we can observe real resource usage and verify that limits are being enforced.

---

## Prerequisites

```bash
oc adm new-project i27-resources-demo
oc project i27-resources-demo
```

Apply SCC (required for the stress image):

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-resources-demo
```

---

## Understanding the Stress Tool

`polinux/stress` runs the Linux `stress` utility which deliberately consumes CPU and memory.

The arguments used in this lab:

| Argument | Value | Meaning |
|----------|-------|---------|
| `--vm` | `1` | Spawn 1 virtual memory worker |
| `--vm-bytes` | `100M` | Each worker allocates 100MB of memory |
| `--vm-hang` | `1` | Keep the allocated memory for 1 second before freeing and re-allocating |

This creates a continuous memory allocation pattern of ~100MB which we can observe against the configured limits.

---

## Deployment YAML

**`resources-demo.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resources-demo
  namespace: i27-resources-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resources-demo
  template:
    metadata:
      labels:
        app: resources-demo
    spec:
      containers:
      - name: stress
        image: polinux/stress
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
        resources:
          requests:
            memory: "100Mi"   # scheduler guarantees this much on the node
            cpu: "50m"        # 50 millicores = 0.05 core guaranteed
          limits:
            memory: "200Mi"   # container can use up to 200Mi, no more
            cpu: "100m"       # container can use up to 100 millicores
```

### Resource Values Explained

| Field | Value | Meaning |
|-------|-------|---------|
| `requests.memory` | `100Mi` | Node must have 100Mi free to schedule this pod |
| `requests.cpu` | `50m` | Node must have 50 millicores free to schedule this pod |
| `limits.memory` | `200Mi` | If container exceeds 200Mi → OOMKilled |
| `limits.cpu` | `100m` | If container exceeds 100m → CPU throttled (not killed) |

The stress tool allocates 100M which fits within the 200Mi memory limit — the pod should run stably.

---

## Apply

```bash
oc apply -f resources-demo.yaml
```

---

## Verify Pod is Running

```bash
oc get pods -n i27-resources-demo
```

Expected:

```
NAME                             READY   STATUS    RESTARTS   AGE
resources-demo-xxxxxxxx-xxxxx    1/1     Running   0          30s
```

---

## Inspect Resource Configuration

```bash
oc describe pod <pod-name> -n i27-resources-demo
```

Look for the `Containers` section:

```
Containers:
  stress:
    Image:    polinux/stress
    Limits:
      cpu:     100m
      memory:  200Mi
    Requests:
      cpu:     50m
      memory:  100Mi
```

---

## Check QoS Class

```bash
oc get pod <pod-name> -n i27-resources-demo -o jsonpath='{.status.qosClass}'
```

Expected output: `Burstable`

Because requests (100Mi / 50m) are less than limits (200Mi / 100m), the pod is classified as `Burstable`.

---

## View Live Resource Usage

```bash
oc adm top pods -n i27-resources-demo
```

Expected output:

```
NAME                             CPU(cores)   MEMORY(bytes)
resources-demo-xxxxxxxx-xxxxx    45m          102Mi
```

The actual CPU usage will be around 45-50m (within the 100m limit) and memory will hover around 100Mi (within the 200Mi limit).

Compare node-level resource usage:

```bash
oc adm top nodes
```

---

## Verify Requests vs Limits in Events

If the pod was scheduled correctly with no resource-related issues, the events section of `oc describe pod` will show no warnings — just a clean `Started` event:

```
Events:
  Normal  Scheduled  Successfully assigned i27-resources-demo/resources-demo-xxx to <node>
  Normal  Pulled     Container image already present on machine
  Normal  Created    Created container stress
  Normal  Started    Started container stress
```
