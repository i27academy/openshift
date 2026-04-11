
---

# 🟧 OpenShift Deployment: Scaling & Image Management

### 1. Initial Setup & Security
OpenShift requires a project and an SCC to run the Nginx image on port 80.

```bash
# Create and switch to project
oc new-project i27-deployment-demo

# Apply SCC (Essential for Nginx)
oc adm policy add-scc-to-user anyuid -z default -n i27-deployment-demo
```

---

### 2. The Deployment Manifest (`nginx-deployment.yaml`)
We use a standard Kubernetes-native **Deployment**. 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-deployment
  annotations:
    kubernetes.io/change-cause: "Initial Blue Version"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-deployment
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
      - name: nginx
        image: devopswithcloudhub/nginx:blue
        ports:
        - containerPort: 80
```

**Apply the Deployment:**
```bash
oc apply -f nginx-deployment.yaml
```

---

### 3. Managing the Deployment

#### **A. Scaling the Deployment**
Scaling adds or removes Pods to handle traffic load.
```bash
oc scale deployment/nginx-deployment --replicas=4
```

> **📝 Note:** > Scaling is handled by the **ReplicaSet**. When you run this command, the Deployment updates the ReplicaSet, which then triggers the creation of 2 additional Pods to meet the new "Desired State."



#### **B. Updating the Image (Rolling Update)**
Updating the image triggers a new **Rollout**.
```bash
# Update the container image
oc set image deployment/nginx-deployment nginx=devopswithcloudhub/nginx:green

# Annotate the change for history tracking
oc annotate deployment nginx-deployment kubernetes.io/change-cause="Updated to Green Version" --overwrite
```

> **📝 Note:** > By default, OpenShift uses the **RollingUpdate** strategy. It will create a *new* ReplicaSet for the "Green" version and gradually transition Pods from the "Blue" ReplicaSet to the "Green" one.

---

### 4. Verification & History
Always verify the status of your rollout and maintain a clean history for rollbacks.

**Check Rollout Status:**
```bash
oc rollout status deployment/nginx-deployment
```

**View Rollout History:**
```bash
oc rollout history deployment/nginx-deployment
```

---

### 5. Summary Comparison Table

| Action | Command | Controller Responsible |
| :--- | :--- | :--- |
| **Scale** | `oc scale` | **ReplicaSet** (Adjusts Pod count) |
| **Update** | `oc set image` | **Deployment** (Creates new ReplicaSet) |
| **Verify** | `oc rollout status` | **Deployment Controller** |

---

### 6. Cleanup
```bash
# Delete the project to wipe all resources
oc delete project i27-deployment-demo
```
