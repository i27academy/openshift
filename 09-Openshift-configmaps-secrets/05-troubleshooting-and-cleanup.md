# OpenShift – ConfigMap and Secret Troubleshooting and Cleanup

## Troubleshooting

### Pod not starting — CreateContainerConfigError

This means the pod found a reference to a ConfigMap or Secret that does not exist.

```bash
oc get pods -n i27-config-demo
oc describe pod <pod-name> -n i27-config-demo
```

Look for events like:

```
Error: configmap "db-config" not found
Error: secret "db-secret" not found
```

Fix: create the missing ConfigMap or Secret before applying the deployment.

```bash
oc get cm -n i27-config-demo
oc get secret -n i27-config-demo
```

---

### Env var is empty or missing

The pod started but `printenv` shows an empty or missing variable.

```bash
oc exec -it <pod-name> -n i27-config-demo -- printenv | grep DATABASE
```

Common causes:

| Cause | Fix |
|-------|-----|
| Wrong key name in `configMapKeyRef` | Check exact key name with `oc describe cm db-config` |
| Wrong key name in `secretKeyRef` | Check exact key name with `oc describe secret db-secret` |
| CM or Secret updated but pod not restarted | Restart the deployment: `oc rollout restart deployment/deploy-env-demo -n i27-config-demo` |
| Typo in `name` field of `configMapRef` or `secretRef` | Verify the CM/Secret name matches exactly |

Verify key names in the ConfigMap:

```bash
oc describe cm db-config -n i27-config-demo
```

Verify key names in the Secret:

```bash
oc describe secret db-secret -n i27-config-demo
```

---

### Volume not mounting — pod stuck in ContainerCreating

```bash
oc describe pod <pod-name> -n i27-config-demo
```

Look for events like:

```
MountVolume.SetUp failed: configmap "db-config" not found
```

Common causes:

| Cause | Fix |
|-------|-----|
| `volumes[].configMap.name` typo | Must match the ConfigMap `metadata.name` exactly |
| `volumes[].secret.secretName` typo | Must match the Secret `metadata.name` exactly |
| `volumeMounts[].name` does not match `volumes[].name` | Both must use the same volume name |

Check the wiring:

```yaml
# volumes[].name must match volumeMounts[].name
volumes:
- name: config-volume        ← this name
  configMap:
    name: db-config

volumeMounts:
- name: config-volume        ← must match exactly
  mountPath: /etc/config
```

---

### Mounted volume files are missing inside pod

```bash
oc exec -it <pod-name> -n i27-config-demo -- ls /etc/config/
```

If the directory is empty or the expected files are missing:

- Verify the ConfigMap has the expected keys: `oc get cm db-config -o yaml -n i27-config-demo`
- If using `items[].key`, ensure the key name matches exactly
- If using `subPath`, the file will only appear at that exact path, not in the directory listing

---

### Updating a Live ConfigMap and Observing the Change

ConfigMaps mounted as **volumes** update automatically inside running pods. Environment variable injections do not — they require a pod restart.

Edit the ConfigMap:

```bash
oc edit cm db-config -n i27-config-demo
```

Change a value, save, and exit. Wait 30–60 seconds (kubelet sync interval), then verify:

```bash
oc exec -it <pod-name> -n i27-config-demo -- cat /etc/config/DB_HOST
```

The updated value will appear without restarting the pod.

To force a restart for env var changes:

```bash
oc rollout restart deployment/deploy-env-demo -n i27-config-demo
oc rollout restart deployment/deploy-volume-demo -n i27-config-demo
```

---

### SCC-related pod failure

If the pod fails to start and describe shows permission errors:

```bash
oc describe pod <pod-name> -n i27-config-demo
```

Look for:

```
container has runAsNonRoot and image will run as root
```

Fix:

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-config-demo
```

Then delete and re-apply the deployment.

---

## Cleanup

Always clean up in this order — delete deployments before ConfigMaps and Secrets to avoid pods referencing deleted resources.

Delete deployments:

```bash
oc delete deployment deploy-env-demo -n i27-config-demo
oc delete deployment deploy-volume-demo -n i27-config-demo
```

Delete ConfigMaps:

```bash
oc delete cm db-config -n i27-config-demo
oc delete cm nginx-config -n i27-config-demo
```

Delete Secrets:

```bash
oc delete secret db-secret -n i27-config-demo
```

Delete the project (removes everything):

```bash
oc delete project i27-config-demo
```

---

## Quick Reference

```bash
# ConfigMap
oc create configmap db-config --from-literal=KEY=value
oc get cm -n i27-config-demo
oc describe cm db-config -n i27-config-demo
oc get cm db-config -o yaml -n i27-config-demo
oc edit cm db-config -n i27-config-demo

# Secret
oc create secret generic db-secret --from-literal=KEY=value
oc get secret -n i27-config-demo
oc describe secret db-secret -n i27-config-demo
oc get secret db-secret -o yaml -n i27-config-demo
oc get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' -n i27-config-demo | base64 -d
oc extract secret/db-secret -n i27-config-demo --to=./extracted/

# Encode / Decode
echo -n 'myvalue' | base64
echo 'bXl2YWx1ZQ==' | base64 -d

# Rollout restart after CM/Secret update
oc rollout restart deployment/<name> -n i27-config-demo
oc rollout status deployment/<name> -n i27-config-demo

# Exec to verify
oc exec -it <pod-name> -n i27-config-demo -- printenv
oc exec -it <pod-name> -n i27-config-demo -- ls /etc/config/
oc exec -it <pod-name> -n i27-config-demo -- cat /etc/config/DB_HOST
```
