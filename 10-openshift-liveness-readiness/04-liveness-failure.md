# OpenShift – Liveness Probe Failure

## What We Are Proving

When the liveness probe fails:

- OpenShift considers the container unhealthy
- The container is **restarted**
- RESTARTS counter **increments**
- After repeated failures, the pod may enter `CrashLoopBackOff`

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

## YAML — Liveness Pointing to Wrong Path

The only change from the passing example is the liveness probe `path` — it now points to `/wrong-live` which does not exist on the app and returns 404.

**`i27-node-probes-liveness-fail.yaml`**

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
            path: /wrong-live    # wrong path — causes liveness failure
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready          # readiness still correct
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
oc apply -f i27-node-probes-liveness-fail.yaml
```

---

## Observe the Failure

Watch pods in real time:

```bash
oc get pods -w
```

Initially the pod starts and readiness passes (correct path). After `initialDelaySeconds` (10s), the liveness probe begins firing against `/wrong-live`. After `failureThreshold` (3) consecutive failures — 15 seconds — the container is restarted.

You will see RESTARTS incrementing:

```
NAME                              READY   STATUS    RESTARTS   AGE
i27-node-probes-xxxxxxx-xxxxx     1/1     Running   0          15s
i27-node-probes-xxxxxxx-xxxxx     0/1     Running   1          35s
i27-node-probes-xxxxxxx-xxxxx     1/1     Running   1          38s
i27-node-probes-xxxxxxx-xxxxx     0/1     Running   2          60s
```

If left running, the pod will eventually enter `CrashLoopBackOff`:

```
NAME                              READY   STATUS             RESTARTS   AGE
i27-node-probes-xxxxxxx-xxxxx     0/1     CrashLoopBackOff   5          4m
```

---

## Confirm in Pod Events

```bash
oc describe pod <pod-name>
```

Look for events at the bottom:

```
Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 404
Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 404
Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 404
Normal   Killing    Container nodeapp failed liveness probe, will be restarted
```

The `Killing` event confirms OpenShift restarted the container due to liveness failure.

---

## Compare with Readiness Failure

| | Readiness Failure | Liveness Failure |
|--|-------------------|-----------------|
| Container restarted? | No | Yes |
| RESTARTS increments? | No | Yes |
| Removed from endpoints? | Yes | Yes (briefly during restart) |
| CrashLoopBackOff possible? | No | Yes |

---

## Fix — Restore Correct Liveness Path

Apply the passing YAML:

```bash
oc apply -f i27-node-probes-pass.yaml
```

Watch pods stabilize:

```bash
oc get pods -w
```

RESTARTS stops incrementing. Pods return to `1/1 READY`. Check endpoints:

```bash
oc get endpoints i27-node-probes-service
```

Test the route:

```bash
curl $ROUTE_URL
```

Everything is restored.

---

## Key Takeaway

Liveness probe controls the **container lifecycle**. A failed liveness probe means OpenShift no longer trusts the container to recover on its own — so it kills and restarts it. Unlike readiness failures, this directly impacts pod stability and can escalate to `CrashLoopBackOff` if the underlying issue is not fixed.
