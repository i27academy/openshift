
---

# 📘 OpenShift Pods – Creation with Labels (OC + YAML)

---

## 🔹 Create Pod with Label (CLI)

```bash
oc run nginx-pod --image=nginx --labels="app=web,env=dev"
```

Verify:

```bash
oc get pods
```

---

## 🔹 Create Pod with Label (Explicit Key=Value)

```bash
oc run api-pod --image=nginx --labels="tier=backend,version=v1"
```

---

## 🔹 View Pod Labels

```bash
oc get pods --show-labels
```

---

## 🔹 Filter Pods Using Label Selector

```bash
oc get pods -l app=web
oc get pods -l env=dev
oc get pods -l tier=backend
```

---

## 🔹 Filter Pods with Multiple Labels (AND condition)

```bash
oc get pods -l app=web,env=dev
```

---

## 🔹 Sort Pods (With Labels Visible)

```bash
oc get pods --sort-by=.metadata.name
oc get pods --sort-by=.metadata.creationTimestamp
```

---

## 🔹 Add Label to Existing Pod

```bash
oc label pod nginx-pod release=stable
```

---

## 🔹 Overwrite Existing Label

```bash
oc label pod nginx-pod env=prod --overwrite
```

---

## 🔹 Remove Label from Pod

```bash
oc label pod nginx-pod env-
```

---

## 🔹 Get Pod as YAML (With Labels)

```bash
oc get pod nginx-pod -o yaml
```

---

## 🔹 Pod YAML with Labels

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-yaml
  labels:
    app: web
    env: dev
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx
```

Apply:

```bash
oc apply -f pod.yaml
```

---

## 🔹 Filter Pods Created via YAML

```bash
oc get pods -l app=web
oc get pods -l tier=frontend
```

---

## 🔹 Describe Pod (Includes Labels)

```bash
oc describe pod nginx-pod-yaml
```

---

## 🔹 Delete Pod Using Label Selector

```bash
oc delete pod -l env=dev
```

---

## 🔹 Watch Pods with Labels

```bash
oc get pods -l app=web -w
```

---

## 🔹 Count Pods Using Labels

```bash
oc get pods -l app=web --no-headers | wc -l
```

---

## 🔹 Compare Kubernetes Equivalent

```bash
kubectl get pods -l app=web
```

---

