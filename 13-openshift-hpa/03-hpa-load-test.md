# OpenShift – HPA Load Test

## What We Are Doing

Generate sustained HTTP load against the deployed app to push CPU utilization above the HPA threshold (50%). Watch the HPA detect the spike and automatically scale replicas up. Then stop the load and watch the HPA scale back down after the cool-down period.

---

## Prerequisites

HPA and deployment must be running from the previous lab.

```bash
oc project i27-hpa-demo
oc get hpa -n i27-hpa-demo
```

Confirm TARGETS shows a real percentage (not `<unknown>`):

```bash
oc get hpa hpa-demo -n i27-hpa-demo
```

Store the route URL:

```bash
ROUTE_URL=http://$(oc get route hpa-demo-route -o jsonpath='{.spec.host}' -n i27-hpa-demo)
```

---

## Step 1 — Start Watching HPA and Pods

Open two terminals before generating load so you can observe scaling in real time.

**Terminal 1 — Watch HPA:**

```bash
oc get hpa hpa-demo -n i27-hpa-demo -w
```

**Terminal 2 — Watch Pods:**

```bash
oc get pods -n i27-hpa-demo -w
```

---

## Step 2 — Generate Load

In a **third terminal**, run a busybox pod that sends continuous HTTP requests to the app:

```bash
oc run load-generator \
  --image=busybox \
  --restart=Never \
  -n i27-hpa-demo \
  -- /bin/sh -c "while true; do wget -q -O- $ROUTE_URL > /dev/null; done"
```

This pod sends requests in a tight loop, driving up CPU utilization on the app pods.

---

## Step 3 — Observe Scale-Up

Watch Terminal 1 (HPA). After a short delay, the TARGETS percentage will climb above 50%:

```
NAME       REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS
hpa-demo   Deployment/hpa-demo   8%/50%     1         5         1
hpa-demo   Deployment/hpa-demo   62%/50%    1         5         1
hpa-demo   Deployment/hpa-demo   85%/50%    1         5         2
hpa-demo   Deployment/hpa-demo   73%/50%    1         5         3
hpa-demo   Deployment/hpa-demo   51%/50%    1         5         3
```

Watch Terminal 2 (Pods). New pods appear as HPA scales up:

```
NAME                        READY   STATUS              REPLICAS
hpa-demo-xxxxxxx-aaaaa      1/1     Running             
hpa-demo-xxxxxxx-bbbbb      0/1     ContainerCreating   
hpa-demo-xxxxxxx-bbbbb      1/1     Running             
hpa-demo-xxxxxxx-ccccc      0/1     ContainerCreating   
hpa-demo-xxxxxxx-ccccc      1/1     Running             
```

---

## Step 4 — Confirm Scale-Up in HPA Events

```bash
oc describe hpa hpa-demo -n i27-hpa-demo
```

Look for scaling events at the bottom:

```
Events:
  Normal  SuccessfulRescale  New size: 2; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  New size: 3; reason: cpu resource utilization (percentage of request) above target
```

---

## Step 5 — Stop Load

Delete the load generator pod:

```bash
oc delete pod load-generator -n i27-hpa-demo
```

---

## Step 6 — Observe Scale-Down

Watch the HPA — CPU usage will drop as load is removed:

```
NAME       REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS
hpa-demo   Deployment/hpa-demo   51%/50%    1         5         3
hpa-demo   Deployment/hpa-demo   18%/50%    1         5         3
hpa-demo   Deployment/hpa-demo   5%/50%     1         5         3
hpa-demo   Deployment/hpa-demo   5%/50%     1         5         2
hpa-demo   Deployment/hpa-demo   4%/50%     1         5         1
```

Scale-down is **slower than scale-up** — OpenShift waits 5 minutes by default before scaling down to avoid thrashing. This is expected behaviour.

Scaling events in describe:

```
Events:
  Normal  SuccessfulRescale  New size: 2; reason: All metrics below target
  Normal  SuccessfulRescale  New size: 1; reason: All metrics below target
```

---

## Step 7 — Final State Verification

Once load is removed and cool-down completes:

```bash
oc get hpa -n i27-hpa-demo
```

```
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS
hpa-demo   Deployment/hpa-demo   4%/50%    1         5         1
```

Replicas back to 1. The HPA has stabilized.

```bash
oc get pods -n i27-hpa-demo
```

Only 1 pod running — the extra replicas were scaled down automatically.

---

## Scale-Up vs Scale-Down Summary

| | Scale-Up | Scale-Down |
|--|----------|-----------|
| Trigger | CPU exceeds target | CPU drops below target |
| Speed | Fast (~15-30 seconds) | Slow (5 minute cool-down) |
| Reason for difference | Protect availability | Prevent thrashing |
| Events visible in | `oc describe hpa` | `oc describe hpa` |
