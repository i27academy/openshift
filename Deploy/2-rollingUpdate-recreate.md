
---

# 🟧 OpenShift Deployment Strategies: Deep Dive


### 🎯 Objective
To master the lifecycle of a Kubernetes-native **Deployment** and understand the trade-offs between **Availability** (RollingUpdate) and **Consistency** (Recreate).

---

## 1. Environment Initialization & Security
OpenShift is "Secure by Default." We must prepare the project and override standard security for our demo images.

```bash
# 1. Create and Switch Project
oc new-project i27-strategy-deepdive
oc project i27-strategy-deepdive

# 2. Apply Security Context Constraint (SCC)
oc adm policy add-scc-to-user anyuid -z default -n i27-strategy-deepdive
```

> **📝 Note:**
> * **Why SCC?** Standard Nginx images try to run as `root` and use port `80`. OpenShift blocks this for security.
> * **The Fix:** The `anyuid` SCC allows our service account to run these specific containers. We will deep-dive into SCC architecture in a future session.

---

## 2. Strategy 1: RollingUpdate (The Production Standard)
**Goal:** Achieve zero-downtime by gradually replacing old Pods with new ones.

### ✅ Theoretical Logic

If we have 4 replicas with `maxSurge: 25%` and `maxUnavailable: 25%`:
1. OpenShift creates **1 extra** new Pod (Total = 5).
2. Once the new Pod is healthy, it terminates **1 old** Pod.
3. It repeats this until all 4 Pods are the new version.

### ✅ YAML Manifest (`deployment-rolling.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: i27-rolling-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  selector:
    matchLabels:
      app: i27-rolling
  template:
    metadata:
      labels:
        app: i27-rolling
    spec:
      containers:
      - name: nginx
        image: devopswithcloudhub/nginx:blue
        ports:
        - containerPort: 80
```

### ✅  Execution
```bash
# Apply and Expose
oc apply -f deployment-rolling.yaml
oc expose deployment i27-rolling-app --port=80
oc expose svc i27-rolling-app

# Trigger Update
oc set image deploy/i27-rolling-app nginx=devopswithcloudhub/nginx:green
```

---

## 3. Strategy 2: Recreate (The Clean Slate)
**Goal:** Ensure that Version 1 and Version 2 **never** run at the same time, accepting brief downtime.

### ✅ Theoretical Logic

1. OpenShift terminates **all** existing Pods (3 $\rightarrow$ 0).
2. Only after all old Pods are gone does it start the new ones (0 $\rightarrow$ 3).
3. Result: Clean cut-over, but the app is offline for a few seconds.

### ✅ YAML Manifest (`deployment-recreate.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: i27-recreate-app
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: i27-recreate
  template:
    metadata:
      labels:
        app: i27-recreate
    spec:
      containers:
      - name: nginx
        image: devopswithcloudhub/nginx:blue
        ports:
        - containerPort: 80
```

### ✅  Execution
```bash
# Apply and Expose
oc apply -f deployment-recreate.yaml
oc expose deployment i27-recreate-app --port=80
oc expose svc i27-recreate-app

# Trigger Update
oc set image deploy/i27-recreate-app nginx=devopswithcloudhub/nginx:green
```

---

## 4. Side-by-Side Comparison Table

| Feature | RollingUpdate | Recreate |
| :--- | :--- | :--- |
| **Downtime** | **Zero / Minimal** | **Yes (Brief)** |
| **Version Overlap** | Yes (Old + New run together) | No (Old dies, then New starts) |
| **Resource Usage** | Temporarily Higher (Surge) | Drops to Zero, then Increases |
| **Best For** | Web Apps, APIs, Microservices | Legacy Apps, DB Migrations |

---

## 5. Post-Demo Cleanup
Always leave your cluster clean after a session.

```bash
# Deleting the project wipes all Deployments, Services, Routes, and Pods
oc delete project i27-strategy-deepdive
```

---

