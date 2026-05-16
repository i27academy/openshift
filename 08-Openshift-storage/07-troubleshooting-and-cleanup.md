# OpenShift Storage — Troubleshooting and Cleanup

## Troubleshooting

### PVC stuck in Pending

This is the most common issue. Check the PVC events first:

```bash
oc describe pvc <pvc-name> -n i27-storage-demo
```

Look at the `Events` section at the bottom of the output.

| Cause | What to check |
|-------|--------------|
| StorageClass not found | `oc get sc` — verify the SC name matches exactly |
| Provisioner not running | `oc get pods -n openshift-cluster-csi-drivers` |
| Filestore API disabled | `gcloud services list --enabled` — look for `file.googleapis.com` |
| `WaitForFirstConsumer` mode | PVC stays Pending until a pod is scheduled — this is expected |

---

### Pod stuck in ContainerCreating

The pod has been scheduled but the volume has not mounted yet.

```bash
oc describe pod <pod-name> -n i27-storage-demo
```

Look for events like:

```
Warning  FailedMount  Unable to attach or mount volumes
```

Common causes:

| Cause | Fix |
|-------|-----|
| PVC is still Pending | Wait for PVC to reach Bound status |
| Wrong PVC name in deployment | Check `claimName` matches the PVC `metadata.name` exactly |
| PVC in different namespace | PVC and Pod must be in the same namespace |

---

### Volume expansion failing

```bash
oc describe pvc <pvc-name> -n i27-storage-demo
```

If you see:

```
only dynamically provisioned PVCs can be resized and the storageclass
that provisions the pvc must support resize
```

The StorageClass does not have `allowVolumeExpansion: true`. Patch it:

```bash
oc patch sc <storageclass-name> -p '{"allowVolumeExpansion": true}'
```

Then retry the PVC expansion.

---

### Filestore PVC mount error — wrong network

If pods show mount errors specific to Filestore:

```bash
oc describe pod <pod-name> -n i27-storage-demo
```

Look for NFS mount failure in events. This usually means the Filestore instance is on a different VPC network than the cluster nodes. Ensure the StorageClass parameter `network: default` matches your cluster's VPC.

---

### Filestore PVC takes long to bind

Filestore instance provisioning on GCP can take 2–5 minutes. The PVC will stay in `Pending` during this time. Watch it:

```bash
oc get pvc -n i27-storage-demo -w
```

This is normal. Wait for it to reach `Bound`.

---

### Selector/label mismatch on service (related)

If endpoints are empty after deploying:

```bash
oc get pods --show-labels -n i27-storage-demo
oc describe svc <service-name> -n i27-storage-demo
```

Ensure the Service `selector` matches the Pod labels exactly.

---

## Cleanup Order

Always clean up in this order to avoid orphaned resources. Deleting a PVC before the deployment can cause the pod to hang.

### Standard storage lab cleanup

```bash
# 1. Delete deployments
oc delete deployment deploy-standard-storage -n i27-storage-demo
oc delete deployment deploy-faster-storage -n i27-storage-demo

# 2. Delete PVCs (triggers PV deletion due to reclaimPolicy: Delete)
oc delete pvc pvc-standard -n i27-storage-demo
oc delete pvc pvc-faster -n i27-storage-demo

# 3. Verify PVs are gone
oc get pv

# 4. Delete StorageClass
oc delete sc faster
```

---

### Filestore lab cleanup

> **Important:** Filestore instances cost money. Always verify deletion in the GCP Console.

```bash
# 1. Delete deployment
oc delete deployment deploy-filestore-rwx -n i27-storage-demo

# 2. Delete PVC (triggers Filestore instance deletion)
oc delete pvc pvc-filestore-rwx -n i27-storage-demo

# 3. Verify PV is gone
oc get pv

# 4. Delete StorageClass
oc delete sc sc-filestore-rwx
```

Verify in GCP Console:

```
Google Cloud Console → Filestore → Instances
```

Confirm no instances remain.

---

### Delete the entire project

If all labs are complete, delete the project to remove everything at once:

```bash
oc delete project i27-storage-demo
```

---

## Quick Reference

```bash
# Check all storage objects
oc get sc
oc get pvc -n i27-storage-demo
oc get pv

# Describe for events and errors
oc describe pvc <name> -n i27-storage-demo
oc describe pod <name> -n i27-storage-demo

# Watch PVC bind status
oc get pvc -n i27-storage-demo -w

# Expand a PVC
oc patch pvc <name> -n i27-storage-demo \
  -p '{"spec":{"resources":{"requests":{"storage":"<new-size>"}}}}'

# Enable volume expansion on SC
oc patch sc <name> -p '{"allowVolumeExpansion": true}'

# Exec into pod to test storage
oc exec -it <pod-name> -n i27-storage-demo -- sh
```
