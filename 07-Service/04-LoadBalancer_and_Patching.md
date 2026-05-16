# OpenShift Services — LoadBalancer and Patching

## What is LoadBalancer?

LoadBalancer provisions an external load balancer through the cloud provider (AWS, GCP, Azure). It assigns a public IP to the Service, making the application reachable from outside the cluster without needing a Route.

This type requires the cluster to be running on a cloud platform with load balancer integration configured.

---

## Prerequisites

Project `i27-service-demo` created and `nginx-app` deployment running. See `01-services-overview.md` for setup steps.

---

## LoadBalancer YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer-service
spec:
  selector:
    app: nginx-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

---

## Apply

```bash
oc apply -f loadbalancer.yaml
```

Verify:

```bash
oc get svc nginx-loadbalancer-service
```

Expected output immediately after creation:

```
NAME                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-loadbalancer-service   LoadBalancer   172.30.88.22    <pending>     80:31xxx/TCP   10s
```

Once the cloud load balancer is provisioned:

```
EXTERNAL-IP = 34.x.x.x
```

`<pending>` is normal in lab environments or when cloud integration is not configured.

---

## Patching an Existing Service

In production, avoid deleting and recreating Services unnecessarily. Instead, use `oc patch` to change the Service type in place.

### Situation

If a ClusterIP Service was already created:

```bash
oc expose deployment nginx-app --port=80
```

Running the same expose command again with a different type will fail:

```
Error from server (AlreadyExists): services "nginx-app" already exists
```

### Solution: Patch the existing Service

Convert to NodePort:

```bash
oc patch svc nginx-app -p '{"spec":{"type":"NodePort"}}'
```

Convert to LoadBalancer:

```bash
oc patch svc nginx-app -p '{"spec":{"type":"LoadBalancer"}}'
```

Convert back to ClusterIP:

```bash
oc patch svc nginx-app -p '{"spec":{"type":"ClusterIP"}}'
```

Verify after each patch:

```bash
oc get svc nginx-app
```

### Alternative: Edit directly

```bash
oc edit svc nginx-app
```

Find `type: ClusterIP` and change it to the desired type. Save and exit.

---

## Service Type Comparison

| Type | Internal Access | External Access | Cloud Required |
|------|----------------|-----------------|----------------|
| ClusterIP | Yes | No | No |
| NodePort | Yes | Via NodeIP:Port | No |
| LoadBalancer | Yes | Via public IP | Yes |
