# OpenShift Services — ClusterIP

## What is ClusterIP?

ClusterIP is the **default** Service type. It assigns the Service a stable internal IP that is only reachable from within the cluster. It is used for Pod-to-Pod communication such as backend APIs, databases, and internal microservices.

---

## Prerequisites

Project `i27-service-demo` created and `nginx-app` deployment running. See `01-services-overview.md` for setup steps.

---

## ClusterIP YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  labels:
    app: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: devopswithcloudhub/nginx:blue
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip-service
spec:
  selector:
    app: nginx-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

---

## Apply

```bash
oc apply -f clusterip.yaml
```

Alternatively, if the deployment already exists, create the Service only using the CLI:

```bash
oc expose deployment nginx-app --port=80 --target-port=80 --name=nginx-clusterip-service
```

---

## Verify the Service

```bash
oc get svc nginx-clusterip-service
```

Expected output:

```
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx-clusterip-service   ClusterIP   172.30.102.15    <none>        80/TCP    10s
```

---

## Test Internal Access

ClusterIP is not reachable from outside the cluster. To test it, create a temporary Pod inside the cluster:

```bash
oc run test-pod --rm -it --image=busybox -- /bin/sh
```

Inside the test Pod, run any of the following:

```sh
# Using short service name (same namespace)
wget -qO- http://nginx-clusterip-service

# Using fully qualified DNS
wget -qO- http://nginx-clusterip-service.i27-service-demo.svc.cluster.local

# Using ClusterIP directly
wget -qO- http://172.30.102.15
```

Expected response: the nginx blue app page.

Exit the test Pod:

```sh
exit
```

---

## Verify Endpoints

Endpoints are the Pod IPs the Service is routing traffic to:

```bash
oc get endpoints nginx-clusterip-service
```

Expected output:

```
NAME                      ENDPOINTS                       AGE
nginx-clusterip-service   10.128.0.10:80,10.128.0.11:80
```

If endpoints are empty, the Service selector likely does not match the Pod labels. Check with:

```bash
oc get pods --show-labels
oc describe svc nginx-clusterip-service
```

---

## Summary

| Access method | Works? |
|---------------|--------|
| From inside the cluster (Pod) | Yes |
| From browser | No |
| From external network | No |
