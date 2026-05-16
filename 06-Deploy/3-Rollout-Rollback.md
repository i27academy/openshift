
---

## 🟧 OpenShift Deployment: Rollout & Rollback (End-to-End Demo)

### 🎯 Objective
Demonstrate the lifecycle of a Kubernetes-native **Deployment** in OpenShift, focusing on zero-downtime updates and instant recovery.

**The Workflow:** `Blue (Stable)` → `Green (Update)` → `Orange (Buggy)` → `Rollback to Green`

---

### 1. Project Initialization
Isolate the demo and ensure you are in the correct context.
```bash
oc new-project i27-rollout-demo
oc project i27-rollout-demo
```

### 2. Security Configuration (Critical Step)
OpenShift is "Secure by Default." Standard Nginx images often try to run as the `root` user or use port `80`, which OpenShift blocks.

> **💡 Note on SCC:** By default, OpenShift runs containers using a randomly assigned high-number UID. We are applying the `anyuid` SCC to the `default` ServiceAccount so our Nginx demo images can run without permission errors. We will dive deeper into SCC architecture in a later session.

```bash
# Grant permission to run as any user ID
oc adm policy add-scc-to-user anyuid -z default -n i27-rollout-demo

# Verification
oc get scc anyuid -o yaml | grep i27-rollout-demo -A5
```

---

### 3. Deploying the "Blue" Version
We use a YAML file to define the desired state and include a `change-cause` annotation for better history tracking.

**`deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: i27-nginx-deployment
  annotations:
    kubernetes.io/change-cause: "Initial deployment: Blue version"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: i27-nginx
  template:
    metadata:
      labels:
        app: i27-nginx
    spec:
      containers:
        - name: nginx-container
          image: devopswithcloudhub/nginx:blue
          ports:
            - containerPort: 80
```

**Execution:**
```bash
oc apply -f deployment.yaml
oc rollout status deployment/i27-nginx-deployment
```

---

### 4. Networking: Service & Route
Expose the application to the outside world.

| Step | Command | Purpose |
| :--- | :--- | :--- |
| **Service** | `oc expose deployment i27-nginx-deployment --port=80` | Internal Load Balancing |
| **Route** | `oc expose svc i27-nginx-deployment` | External URL (DNS) |

**Verify Access:**
```bash
ROUTE_URL=$(oc get route i27-nginx-deployment -o jsonpath='{.spec.host}')
curl http://$ROUTE_URL
# Output: BLUE VERSION
```

---

### 5. The Rollout Strategy (Green & Orange)
We will now update the image. Watch how OpenShift performs a **Rolling Update** (killing old pods only after new ones are ready).



**Update to Green:**
```bash
oc set image deployment/i27-nginx-deployment nginx-container=devopswithcloudhub/nginx:green
oc annotate deployment i27-nginx-deployment kubernetes.io/change-cause="Updated to Green version" --overwrite
```

**Update to Orange:**
```bash
oc set image deployment/i27-nginx-deployment nginx-container=devopswithcloudhub/nginx:orange
oc annotate deployment i27-nginx-deployment kubernetes.io/change-cause="Updated to Orange version" --overwrite
```

---

### 6. The Rollback (The "Panic" Button)
Imagine the **Orange** version has a bug. We need to revert to **Green** (Revision 2) immediately.

**Check History:**
```bash
oc rollout history deployment/i27-nginx-deployment
```

**Perform Rollback:**
```bash
# Roll back to the specific stable revision
oc rollout undo deployment/i27-nginx-deployment --to-revision=2
```

**Verify:**
```bash
curl http://$ROUTE_URL
# Output: GREEN VERSION
```

---

### 7. Summary & Best Practices
* **Deployment vs Pods:** Never deploy a Pod directly. Use a **Deployment** to ensure self-healing and scalability.
* **Annotations:** Always use `change-cause`. Without it, `rollout history` just shows "None," making rollbacks a guessing game.
* **SCC:** Remember that `anyuid` is a "quick fix" for demos. In production, we prefer modifying the image to run as a non-root user.



---

### 🛠 Quick Reference Commands
* **Monitor Status:** `oc rollout status deployment/i27-nginx-deployment`
* **Scale Up:** `oc scale deployment i27-nginx-deployment --replicas=5`
* **Describe Pods:** `oc describe pod <pod_name>` (Useful if SCC fails)

