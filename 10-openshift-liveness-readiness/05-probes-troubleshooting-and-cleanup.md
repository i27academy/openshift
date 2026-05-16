# OpenShift – Liveness and Readiness Probes — Troubleshooting and Cleanup

## Troubleshooting

### Pod RESTARTS keep incrementing — CrashLoopBackOff

Liveness probe is failing repeatedly. Most common cause is `initialDelaySeconds` being too low — the probe fires before the app has finished starting up.

```bash
oc get pods
oc describe pod <pod-name>
```

Look for:

```
Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 404
Normal   Killing    Container nodeapp failed liveness probe, will be restarted
```

**Fix:** Increase `initialDelaySeconds` so the probe waits for the app to fully start:

```yaml
livenessProbe:
  initialDelaySeconds: 30   # increase this
```

---

### Pod stuck at 0/1 READY — never receives traffic

Readiness probe is failing. The pod is running but OpenShift is not sending traffic to it.

```bash
oc get pods
oc get endpoints i27-node-probes-service
```

If endpoints show `<none>`:

```bash
oc describe pod <pod-name>
```

Look for:

```
Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 404
```

Common causes:

| Cause | Fix |
|-------|-----|
| Wrong probe path | Verify with `oc exec -- curl http://localhost:3000/ready` |
| App not listening on the specified port | Check `containerPort` matches probe `port` |
| `initialDelaySeconds` too low | Increase to allow app to initialize |
| `failureThreshold` too low | Increase to tolerate transient slow responses |

Manually test the endpoint from inside the pod:

```bash
oc exec -it <pod-name> -- curl -v http://localhost:3000/ready
```

If this returns 200, the path is correct — the issue is with timing. If it returns 404, the path is wrong.

---

### Route shows "Application is not available"

This means the Service has no healthy endpoints — all pods failed their readiness probe.

```bash
oc get endpoints i27-node-probes-service
oc get pods
oc describe pod <pod-name>
```

Fix the readiness probe path and re-apply the correct YAML:

```bash
oc apply -f i27-node-probes-pass.yaml
```

---

### Probe timeout causing false failures

If the app takes longer than `timeoutSeconds` to respond, the probe times out and counts as a failure.

```bash
oc describe pod <pod-name>
```

```
Warning  Unhealthy  Liveness probe failed: (Timeout exceeded while awaiting headers)
```

**Fix:** Increase `timeoutSeconds`:

```yaml
livenessProbe:
  timeoutSeconds: 5
```

---

### Named port missing — Route not mapping correctly

If the Service port is not named and the Route uses a named reference, the mapping fails.

Always name Service ports when using a Route:

```yaml
# Service
ports:
- name: http           # name is required
  port: 80
  targetPort: 3000

# Route
port:
  targetPort: http     # references the name, not the number
```

---

## Recommended Production Probe Values

Tune these based on your app's actual startup time and response characteristics:

```yaml
livenessProbe:
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5

readinessProbe:
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
  successThreshold: 1
  timeoutSeconds: 3
```

---

## Quick Diagnostic Commands

```bash
# Check pod status and restart count
oc get pods

# Watch pods in real time
oc get pods -w

# View probe config and failure events
oc describe pod <pod-name>

# Check Service endpoints
oc get endpoints i27-node-probes-service

# Manually test liveness endpoint from inside pod
oc exec -it <pod-name> -- curl -v http://localhost:3000/live

# Manually test readiness endpoint from inside pod
oc exec -it <pod-name> -- curl -v http://localhost:3000/ready

# View pod logs
oc logs <pod-name>

# Follow logs in real time
oc logs -f <pod-name>

# Store route URL
ROUTE_URL=http://$(oc get route i27-node-probes-route -o jsonpath='{.spec.host}')

# Test via route
curl $ROUTE_URL
curl $ROUTE_URL/live
curl $ROUTE_URL/ready
```

---

## Cleanup

```bash
oc delete project i27-probes-demo
```
