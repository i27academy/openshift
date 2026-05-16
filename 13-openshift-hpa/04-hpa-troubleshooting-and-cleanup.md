# OpenShift – HPA — Troubleshooting and Cleanup

## Troubleshooting

### TARGETS shows `<unknown>/50%`

HPA cannot retrieve metrics for the pods. This is the most common issue when first setting up HPA.

```bash
oc get hpa -n i27-hpa-demo
```

```
NAME       REFERENCE             TARGETS          MINPODS   MAXPODS   REPLICAS
hpa-demo   Deployment/hpa-demo   <unknown>/50%    1         5         1
```

Common causes:

| Cause | Fix |
|-------|-----|
| Metrics Server not running | `oc adm top pods -n i27-hpa-demo` — if this fails, Metrics Server is unavailable |
| Resource requests not set on container | Add `resources.requests.cpu` to the Deployment — HPA cannot calculate % without a request value |
| Metrics not populated yet | Wait 30–60 seconds after pod startup and check again |

Verify resource requests are set:

```bash
oc describe deployment hpa-demo -n i27-hpa-demo | grep -A5 Requests
```

If no requests are shown, add them to the deployment:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
```

```bash
oc rollout restart deployment/hpa-demo -n i27-hpa-demo
```

---

### HPA not scaling up — already at maxReplicas

Load is high but replicas are not increasing.

```bash
oc get hpa -n i27-hpa-demo
```

```
NAME       REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS
hpa-demo   Deployment/hpa-demo   95%/50%    1         5         5
```

REPLICAS equals MAXPODS — HPA has reached its ceiling and cannot scale further.

**Fix:** Increase `maxReplicas`:

```bash
oc patch hpa hpa-demo -n i27-hpa-demo \
  -p '{"spec":{"maxReplicas": 10}}'
```

Or edit the HPA directly:

```bash
oc edit hpa hpa-demo -n i27-hpa-demo
```

---

### HPA not scaling down — replicas staying high after load removed

This is expected behaviour. OpenShift has a default **5-minute cool-down period** before scaling down to prevent thrashing.

```bash
oc describe hpa hpa-demo -n i27-hpa-demo
```

Check the events — if you see `All metrics below target` but no scale-down event yet, the cool-down is still in progress. Wait and watch:

```bash
oc get hpa hpa-demo -n i27-hpa-demo -w
```

Scale-down will happen automatically once the stabilization window passes.

---

### HPA created but deployment not scaling — missing scaleTargetRef

If the HPA was created but the deployment never scales, verify the `scaleTargetRef` name matches the actual deployment name exactly:

```bash
oc describe hpa hpa-demo -n i27-hpa-demo
```

Look for:

```
Reference:  Deployment/hpa-demo
```

If the deployment name is different, delete and recreate the HPA with the correct name:

```bash
oc delete hpa hpa-demo -n i27-hpa-demo
oc autoscale deployment <correct-deployment-name> --min=1 --max=5 --cpu-percent=50 -n i27-hpa-demo
```

---

### Load generator pod not generating enough load

If TARGETS stays low even with the load generator running, the app may be responding too fast and not consuming enough CPU.

Check load generator is actually running:

```bash
oc get pods -n i27-hpa-demo
oc logs load-generator -n i27-hpa-demo
```

Run multiple load generators in parallel to increase load:

```bash
oc run load-generator-2 \
  --image=busybox \
  --restart=Never \
  -n i27-hpa-demo \
  -- /bin/sh -c "while true; do wget -q -O- $ROUTE_URL > /dev/null; done"
```

---

## Quick Diagnostic Commands

```bash
# Check HPA status and current targets
oc get hpa -n i27-hpa-demo

# Watch HPA in real time
oc get hpa hpa-demo -n i27-hpa-demo -w

# Full HPA details and scaling events
oc describe hpa hpa-demo -n i27-hpa-demo

# Watch pods scaling up/down
oc get pods -n i27-hpa-demo -w

# Check live CPU usage
oc adm top pods -n i27-hpa-demo

# Verify resource requests on deployment
oc describe deployment hpa-demo -n i27-hpa-demo | grep -A5 Requests

# Patch maxReplicas
oc patch hpa hpa-demo -n i27-hpa-demo -p '{"spec":{"maxReplicas": 10}}'

# Edit HPA directly
oc edit hpa hpa-demo -n i27-hpa-demo
```

---

## Cleanup

Delete the load generator if still running:

```bash
oc delete pod load-generator -n i27-hpa-demo
oc delete pod load-generator-2 -n i27-hpa-demo
```

Delete HPA:

```bash
oc delete hpa hpa-demo -n i27-hpa-demo
```

Delete deployment, service, and route:

```bash
oc delete deployment hpa-demo -n i27-hpa-demo
oc delete svc hpa-demo-svc -n i27-hpa-demo
oc delete route hpa-demo-route -n i27-hpa-demo
```

Delete the project:

```bash
oc delete project i27-hpa-demo
```
