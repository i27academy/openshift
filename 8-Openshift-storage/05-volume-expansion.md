# OpenShift Storage — Volume Expansion

## What is Volume Expansion?

Volume expansion allows you to increase the size of a PVC after it has been created and bound. This is done by editing the PVC's `spec.resources.requests.storage` field — no pod restart is required in most cases.

For expansion to work, the StorageClass must have `allowVolumeExpansion: true`.

---

## Expansion with the Default StorageClass

The GKE default StorageClass (`standard`) has `allowVolumeExpansion: true` enabled by default. Expanding a PVC that uses this SC works without any changes.

### Expand `pvc-standard` from 30Gi to 50Gi

```bash
oc patch pvc pvc-standard -n i27-storage-demo \
  -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
```

Verify:

```bash
oc get pvc pvc-standard -n i27-storage-demo
```

Expected output shows the new size:

```
NAME           STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
pvc-standard   Bound    pvc-xxx    50Gi       RWO            standard
```

Check full details:

```bash
oc describe pvc pvc-standard -n i27-storage-demo
```

---

## Expansion with the Custom StorageClass — Fails

The `faster` StorageClass was created without `allowVolumeExpansion`. Attempting to expand `pvc-faster` will fail.

### Attempt to expand `pvc-faster` from 20Gi to 40Gi

```bash
oc patch pvc pvc-faster -n i27-storage-demo \
  -p '{"spec":{"resources":{"requests":{"storage":"40Gi"}}}}'
```

You will see an error similar to:

```
Error from server: persistentvolumeclaims "pvc-faster" is forbidden:
only dynamically provisioned pVCs can be resized and the storageclass
that provisions the pvc must support resize
```

This happens because the `faster` StorageClass does not have `allowVolumeExpansion: true`.

---

## Fix — Enable Volume Expansion on the StorageClass

StorageClass is an immutable resource in most fields, but `allowVolumeExpansion` can be patched directly.

### Option 1 — Patch the existing StorageClass

```bash
oc patch sc faster \
  -p '{"allowVolumeExpansion": true}'
```

Verify:

```bash
oc get sc faster
```

Expected output:

```
NAME     PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION
faster   kubernetes.io/gce-pd   Delete          Immediate           true
```

### Option 2 — Update the YAML and re-apply

Update `storageclass-faster.yaml` to include `allowVolumeExpansion`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: faster
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
allowVolumeExpansion: true
```

Apply:

```bash
oc apply -f storageclass-faster.yaml
```

---

## Retry Expansion After Fix

Now retry the expansion on `pvc-faster`:

```bash
oc patch pvc pvc-faster -n i27-storage-demo \
  -p '{"spec":{"resources":{"requests":{"storage":"40Gi"}}}}'
```

Verify:

```bash
oc get pvc pvc-faster -n i27-storage-demo
```

Expected output:

```
NAME         STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
pvc-faster   Bound    pvc-yyy    40Gi       RWO            faster
```

---

## Important Rules for Volume Expansion

- You can only **increase** storage size, never decrease it
- The StorageClass must have `allowVolumeExpansion: true`
- For filesystem resize to take effect inside the pod, some storage backends require a pod restart
- Block storage (GCP PD) typically resizes online without a restart
