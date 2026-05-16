# OpenShift – Exceeding Resource Limits

## What We Are Proving

When a container exceeds its configured limits, OpenShift responds differently depending on the resource:

- **Memory exceeded** → Container is killed immediately with exit code `137` (OOMKilled)
- **CPU exceeded** → Container is throttled (slowed down), never killed

Both scenarios are demonstrated in this lab using `polinux/stress`.

---

## Prerequisites

```bash
oc project i27-resources-demo
```

---

## Scenario 1 — OOMKill (Memory Limit Exceeded)

### What We Are Doing

Set the memory limit lower than what the stress tool tries to allocate. The container will exceed the limit and be killed by the OOM (Out of Memory) killer.

The stress tool is configured to allocate `150M` but the memory limit is set to `100Mi` — the container will be OOMKilled.

### YAML

**`resources-oomkill.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resources-oomkill
  namespace: i27-resources-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resources-oomkill
  template:
    metadata:
      labels:
        app: resources-oomkill
    spec:
      containers:
      - name: stress
        image: polinux/stress
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
        resources:
          requests:
            memory: "50Mi"
            cpu: "50m"
          limits:
            memory: "100Mi"   # limit is 100Mi but stress allocates 150M → OOMKill
            cpu: "100m"
```

### Apply

```bash
oc apply -f resources-oomkill.yaml
```

### Observe OOMKill

Watch the pod:

```bash
oc get pods -n i27-resources-demo -w
```

You will see the pod start, then get killed and restart repeatedly:

```
NAME                              READY   STATUS      RESTARTS   AGE
resources-oomkill-xxxxxxx-xxxxx   0/1     OOMKilled   0          5s
resources-oomkill-xxxxxxx-xxxxx   0/1     Running     1          8s
resources-oomkill-xxxxxxx-xxxxx   0/1     OOMKilled   1          13s
resources-oomkill-xxxxxxx-xxxxx   0/1     Running     2          16s
```

### Confirm OOMKill in Pod Description

```bash
oc describe pod <pod-name> -n i27-resources-demo
```

Look for the `Last State` and exit code in the containers section:

```
Containers:
  stress:
    Last State:  Terminated
      Reason:    OOMKilled
      Exit Code: 137
```

Exit code `137` always means the container was killed due to exceeding the memory limit.

---

## Scenario 2 — CPU Throttling (CPU Limit Exceeded)

### What We Are Doing

Set the CPU limit very low while the stress tool tries to consume more. Unlike memory, exceeding the CPU limit does not kill the container — it is throttled (artificially slowed down).

### YAML

**`resources-cpu-throttle.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resources-cpu-throttle
  namespace: i27-resources-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resources-cpu-throttle
  template:
    metadata:
      labels:
        app: resources-cpu-throttle
    spec:
      containers:
      - name: stress
        image: polinux/stress
        command: ["stress"]
        args: ["--cpu", "2", "--vm", "1", "--vm-bytes", "50M", "--vm-hang", "1"]
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "50m"     # very low CPU limit — stress will be throttled
```

### Apply

```bash
oc apply -f resources-cpu-throttle.yaml
```

### Observe CPU Throttling

```bash
oc get pods -n i27-resources-demo
```

The pod will be `Running` and **not** restarting — CPU throttling does not kill the container.

Check actual CPU usage vs the limit:

```bash
oc adm top pods -n i27-resources-demo
```

You will see CPU capped at or below `50m` even though stress is trying to use more. The container is being throttled by the kernel cgroup.

---

## Memory vs CPU Limit Behaviour — Summary

| | Memory Limit Exceeded | CPU Limit Exceeded |
|--|----------------------|-------------------|
| Container killed? | Yes — OOMKilled | No |
| Exit code | 137 | N/A |
| RESTARTS increments? | Yes | No |
| Status shown | OOMKilled | Running |
| Recovery | Deployment restarts container | Automatic (just throttled) |
| How to detect | `oc describe pod` → Exit Code 137 | `oc adm top pods` → CPU capped |

---

## Fix OOMKill

Increase the memory limit so it is higher than what the stress tool allocates:

```yaml
limits:
  memory: "200Mi"   # higher than the 150M stress allocates
```

Or reduce the stress allocation to fit within the limit:

```yaml
args: ["--vm", "1", "--vm-bytes", "80M", "--vm-hang", "1"]
```

Re-apply and verify the pod stays in `Running` state without restarting.
