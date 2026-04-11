# 🟧 i27Academy: ReplicaSets & The "Security First" Reality

## 1. The Evolution: From Pods to ReplicaSets
In our previous session, we created standalone Pods. However, Pods alone are not production-ready.

### ❌ The Problem with Standalone Pods
* **No Self-Healing:** If a Pod crashes or a Node fails, the application stays down.
* **Temporary:** Pods are ephemeral; they are not designed to restart themselves.
* **Manual Scaling:** You cannot easily manage 10 identical Pods manually.

### ✅ The ReplicaSet Solution
A **ReplicaSet** ensures that a **desired number** of Pods are running at all times.
* **Self-Healing:** If a Pod is deleted, the ReplicaSet immediately creates a new one.
* **High Availability:** Distributes Pods across the cluster.
* **Scaling:** Easily scale from 1 to 100 Pods with a single command.



---

## 2. Architecture: Labels & Selectors
The ReplicaSet doesn't "own" Pods by name; it finds them using **Labels**.

* **Labels:** Key-value pairs attached to Pods (e.g., `app: nginx`).
* **Selector:** The ReplicaSet’s "search filter." It manages any Pod that matches its defined label.

**Visual Logic:**
```text
ReplicaSet (Selector: app=i27-web)
        │
        ├── Pod 1 (Label: app=i27-web)  👉 MANAGED
        ├── Pod 2 (Label: app=i27-web)  👉 MANAGED
        └── Pod 3 (Label: app=db)       👉 IGNORED
```

---

## 3. Hands-On: Creating an i27 ReplicaSet
**`i27-rs.yaml`**
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: i27-nginx-rs
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
      - name: nginx
        image: nginx:latest
```

**Commands:**
```bash
# Apply the configuration
oc apply -f i27-rs.yaml

# Check the status
oc get rs

# Show labels to see how they are linked
oc get pods --show-labels
```

---

## 4. 🔥 The "Real-World" Failure Scenario (The SCC Trap)
After applying the YAML above, you will notice something strange in OpenShift that doesn't happen in standard Kubernetes.

### The Observation
```bash
oc get pods
```
**Status:** `CrashLoopBackOff`

### The Logs (The Evidence)
```bash
oc logs <pod-name>
```
**Error:** `mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)`

### The Root Cause: SCC (Security Context Constraints)
OpenShift is **Secure by Default**. It uses SCCs to control what a container can do.

> [!NOTE]
>  We are identifying SCC as the root cause here to understand the failure. We will dive deep into the architecture, types, and management of **Security Context Constraints (SCC)** in a later stage of the course.

* **The Conflict:** The standard `nginx` image wants to run as **Root** (UID 0).
* **The Restriction:** OpenShift’s `restricted-v2` SCC forces containers to run as a **Random High-Number UID**.



### 🔐 SCC Comparison Proof
Compare a standalone Pod (created via `oc run`) vs. a Pod created by a ReplicaSet:

| Method | SCC Applied | Result |
| :--- | :--- | :--- |
| `oc run siva-pod` | `anyuid` (often default for users) | ✅ Runs |
| `oc apply -f rs.yaml` | `restricted-v2` | ❌ Fails |

---

## 5. How to Fix the Failure
### Option A: Use an OpenShift-Safe Image (Best Practice)
Use an image designed to run without root privileges.
```bash
# Update your YAML image to:
image: nginxinc/nginx-unprivileged
```

### Option B: Grant `anyuid` Permissions (Lab/Demo Only)
If you must use the standard image, grant the ServiceAccount permission to bypass the restriction.
```bash
oc adm policy add-scc-to-user anyuid -z default -n <your-project>

# Delete the failing pods to let the ReplicaSet recreate them
oc delete pods -l app=i27-nginx
```

---

## 6. Limitations: Why ReplicaSet is Still Not Enough
Even though ReplicaSets provide self-healing, they have major gaps:
* ❌ **No Rolling Updates:** You cannot update an image version without downtime.
* ❌ **No Rollbacks:** If an update fails, there is no easy "undo" button.
* ❌ **No History:** You cannot see previous versions of your application.

**Because of these limits, we move to the final evolution: Deployments.**

---

## 7. Summary & Learning Flow
> **"ReplicaSet didn’t break Nginx. ReplicaSet exposed that the Nginx image was not OpenShift-safe."**

**The i27Academy Progression:**
Projects $\rightarrow$ Pods $\rightarrow$ **ReplicaSets** $\rightarrow$ Deployments (Next Session)

---

### ⌨️ Useful Commands Cheat Sheet
* **Scale up/down:** `oc scale rs i27-nginx-rs --replicas=5`
* **Delete RS:** `oc delete rs i27-nginx-rs` 
* **Self-Healing Test:** `oc delete pod <pod-name>` and watch a new one appear instantly.