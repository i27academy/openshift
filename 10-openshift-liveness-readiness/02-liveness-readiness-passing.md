# OpenShift – Liveness and Readiness Probes — Passing Example

## What We Are Doing

Deploy `i27devopsb8/node:livereadv1` with both probes pointing to correct endpoints. Verify the app is healthy, endpoints are populated, and the route is accessible.

This is the baseline — a correctly configured deployment that we will break in the next labs.

---

## Prerequisites

```bash
oc project i27-probes-demo
```

Apply SCC (required for this image):

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-probes-demo
```

---

## YAML — Deployment, Service, and Route

**`i27-node-probes-pass.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: i27-node-probes
spec:
  replicas: 2
  selector:
    matchLabels:
      app: i27-node-probes
  template:
    metadata:
      labels:
        app: i27-node-probes
    spec:
      containers:
      - name: nodeapp
        image: i27devopsb8/node:livereadv1
        ports:
        - containerPort: 3000
        livenessProbe:
          httpGet:
            path: /live          # correct liveness endpoint
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready         # correct readiness endpoint
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  name: i27-node-probes-service
spec:
  selector:
    app: i27-node-probes
  ports:
  - name: http                   # named port — required for Route mapping
    protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: i27-node-probes-route
spec:
  to:
    kind: Service
    name: i27-node-probes-service
  port:
    targetPort: http             # references the named port above
```

### Why the Named Port?

The Service port is named `http` and the Route references it by that name (`targetPort: http`). This is important in OpenShift — always name Service ports when using a Route. Without a named port, route mapping can fail or become unclear.

```
Route → targetPort: http → Service port name: http → targetPort: 3000 → Pod
```

---

## Apply

```bash
oc apply -f i27-node-probes-pass.yaml
```

---

## Verify

Check pods — both should show `1/1 READY`:

```bash
oc get pods
```

Expected:

```
NAME                              READY   STATUS    RESTARTS   AGE
i27-node-probes-xxxxxxx-xxxxx     1/1     Running   0          30s
i27-node-probes-xxxxxxx-yyyyy     1/1     Running   0          30s
```

Check service:

```bash
oc get svc
```

Check endpoints — both pod IPs should appear:

```bash
oc get endpoints i27-node-probes-service
```

Expected:

```
NAME                      ENDPOINTS                           AGE
i27-node-probes-service   10.128.0.10:3000,10.128.0.11:3000   30s
```

Check route:

```bash
oc get route
```

---

## Test via Route

Store the route URL:

```bash
ROUTE_URL=http://$(oc get route i27-node-probes-route -o jsonpath='{.spec.host}')
```

Test the main app:

```bash
curl $ROUTE_URL
```

Test liveness endpoint:

```bash
curl $ROUTE_URL/live
```

Test readiness endpoint:

```bash
curl $ROUTE_URL/ready
```

All three should return `200` responses confirming the app is healthy and serving traffic.

---

## Confirm Probes in Pod Description

```bash
oc describe pod <pod-name>
```

Look for:

```
Liveness:   http-get http://:3000/live  delay=10s timeout=2s period=5s #success=1 #failure=3
Readiness:  http-get http://:3000/ready delay=5s  timeout=2s period=5s #success=1 #failure=3
```

This confirms both probes are registered and running correctly.
