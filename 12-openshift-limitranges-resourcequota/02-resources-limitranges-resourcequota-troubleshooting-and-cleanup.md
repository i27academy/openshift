# OpenShift – Resource Management — Troubleshooting and Cleanup

## Troubleshooting

### Pod OOMKilled — exit code 137

The container exceeded its memory limit and was killed by the OOM killer.

```bash
oc get pods -n i27-resources-demo
```

Status shows `OOMKilled` and RESTARTS incrementing:

```
NAME                              READY   STATUS      RESTARTS
resources-oomkill-xxxxxxx-xxxxx   0/1     OOMKilled   3
```

Confirm:

```bash
oc describe pod <pod-name> -n i27-resources-demo
```

Look for:

```
Last State:  Terminated
  Reason:    OOMKilled
  Exit Code: 137
```

**Fix options:**

Increase the memory limit:

```yaml
limits:
  memory: "300Mi"
```

Or reduce the memory the app allocates:

```yaml
args: ["--vm", "1", "--vm-bytes", "80M", "--vm-hang", "1"]
```

---

### Pod stuck in Pending — quota exceeded

A new pod cannot be scheduled because the namespace ResourceQuota is full.

```bash
oc get pods -n i27-resources-demo
oc describe replicaset <rs-name> -n i27-resources-demo
```

Look for:

```
Warning  FailedCreate  pods is forbidden: exceeded quota: i27-resourcequota,
requested: requests.memory=100Mi, used: requests.memory=512Mi, limited: requests.memory=512Mi
```

**Fix options:**

Check current quota usage:

```bash
oc describe resourcequota i27-resourcequota -n i27-resources-demo
```

Either scale down existing deployments to free up quota:

```bash
oc scale deployment resources-demo --replicas=1 -n i27-resources-demo
```

Or increase the quota limits:

```bash
oc edit resourcequota i27-resourcequota -n i27-resources-demo
```

---

### Pod stuck in Pending — insufficient resources on node

The node does not have enough free resources to satisfy the pod's requests.

```bash
oc describe pod <pod-name> -n i27-resources-demo
```

Look for:

```
Warning  FailedScheduling  0/3 nodes are available:
  3 Insufficient memory, 3 Insufficient cpu
```

**Fix options:**

Reduce the pod's requests to fit on available nodes:

```yaml
requests:
  memory: "64Mi"
  cpu: "25m"
```

Or check node capacity and allocatable resources:

```bash
oc adm top nodes
oc describe node <node-name>
```

Look for `Allocatable` vs `Requests` in the describe output to see how much is still available.

---

### Pod rejected by LimitRange — exceeds max or below min

```bash
oc apply -f my-deployment.yaml
```

Error:

```
Error: pods is forbidden: maximum memory usage per Container is 512Mi, but limit is 1Gi
```

Or:

```
Error: pods is forbidden: minimum cpu usage per Container is 50m, but request is 10m
```

**Fix:** Update the pod resource spec to fall within the LimitRange bounds:

```bash
oc describe limitrange i27-limitrange -n i27-resources-demo
```

Adjust the deployment YAML to stay within the `min` and `max` values shown.

---

### CPU throttling — app running slowly

The container is hitting its CPU limit and being throttled by the kernel. The pod stays `Running` but responds slowly.

```bash
oc adm top pods -n i27-resources-demo
```

If CPU usage is consistently at or near the limit, the container is being throttled.

**Fix:** Increase the CPU limit:

```yaml
limits:
  cpu: "200m"
```

---

## Quick Diagnostic Commands

```bash
# Check pod status and restarts
oc get pods -n i27-resources-demo

# View resource config and events
oc describe pod <pod-name> -n i27-resources-demo

# Check QoS class
oc get pod <pod-name> -n i27-resources-demo -o jsonpath='{.status.qosClass}'

# Live resource usage — pods
oc adm top pods -n i27-resources-demo

# Live resource usage — nodes
oc adm top nodes

# Check quota usage
oc describe resourcequota i27-resourcequota -n i27-resources-demo

# Check limitrange
oc describe limitrange i27-limitrange -n i27-resources-demo

# Check node allocatable vs used
oc describe node <node-name>
```

---

## Cleanup

Delete individual deployments:

```bash
oc delete deployment resources-demo -n i27-resources-demo
oc delete deployment resources-oomkill -n i27-resources-demo
oc delete deployment resources-cpu-throttle -n i27-resources-demo
oc delete deployment no-resources-demo -n i27-resources-demo
```

Delete LimitRange and ResourceQuota:

```bash
oc delete limitrange i27-limitrange -n i27-resources-demo
oc delete resourcequota i27-resourcequota -n i27-resources-demo
```

Delete the project:

```bash
oc delete project i27-resources-demo
```
