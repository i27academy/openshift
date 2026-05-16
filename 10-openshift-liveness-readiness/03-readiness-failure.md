# OpenShift – Readiness Probe Failure

## What We Are Proving

When the readiness probe fails:

- The pod keeps running
- The container is **not restarted**
- The pod is **removed from Service endpoints**
- Traffic stops reaching that pod
- The Route returns "Application is not available"

RESTARTS counter does **not** increment.

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

## YAML — Readiness Pointing to Wrong Path

The only change from the passing example is the readiness probe `path` — it now points to `/wrong-ready` which does not exist on the app and returns 404.

**`i27-node-probes-readiness-fail.yaml`**

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
            path: /live           # liveness still correct
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /wrong-ready    # wrong path — causes readiness failure
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
  - name: http
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
    targetPort: http
```

---

## Apply

```bash
oc apply -f i27-node-probes-readiness-fail.yaml
```

---

## Observe the Failure

Check pods — READY will show `0/1`:

```bash
oc get pods
```

Expected:

```
NAME                              READY   STATUS    RESTARTS   AGE
i27-node-probes-xxxxxxx-xxxxx     0/1     Running   0          20s
i27-node-probes-xxxxxxx-yyyyy     0/1     Running   0          20s
```

The pod is `Running` but `0/1` — it is alive but not ready. Note **RESTARTS is 0** — no restart happened.

---

## Check Endpoints

```bash
oc get endpoints i27-node-probes-service
```

Expected:

```
NAME                      ENDPOINTS   AGE
i27-node-probes-service   <none>      30s
```

Both pods were removed from the Service endpoints. No traffic is being sent to them.

---

## Check Route

```bash
curl $ROUTE_URL
```

Expected:

```
Application is not available
```

The route has no healthy pods to forward traffic to.

---

## Confirm in Pod Events

```bash
oc describe pod <pod-name>
```

Look for events at the bottom:

```
Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 404
Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 404
Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 404
```

No `Killing` event — the container was never restarted.

---

## Fix — Restore Correct Readiness Path

Apply the passing YAML:

```bash
oc apply -f i27-node-probes-pass.yaml
```

Watch pods recover:

```bash
oc get pods -w
```

Pods will move back to `1/1 READY`. Check endpoints:

```bash
oc get endpoints i27-node-probes-service
```

Pod IPs reappear. Test the route:

```bash
curl $ROUTE_URL
```

Traffic is restored.

---

## Key Takeaway

Readiness probe controls **traffic**, not the container lifecycle. The app was running the entire time — OpenShift simply stopped sending requests to it. Once the probe passes again, traffic resumes automatically with no manual intervention.
