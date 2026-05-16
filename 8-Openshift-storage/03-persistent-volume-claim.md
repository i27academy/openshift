# OpenShift Storage — PersistentVolumeClaim

## What is a PVC?

A PersistentVolumeClaim is a request for storage. A pod does not reference a PV directly — it references a PVC, and the cluster handles finding or creating the matching PV.

When a PVC is created:

1. The cluster looks at the `storageClassName` field
2. The StorageClass provisioner creates a PV automatically
3. The PVC binds to that PV
4. Status changes from `Pending` to `Bound`

---

## PVC with Default StorageClass

When `storageClassName` is omitted, the cluster's default StorageClass is used automatically.

**`pvc.yaml` — pvc-standard**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-standard
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
```

No `storageClassName` field — the GKE default (standard HDD) is used.

Apply:

```bash
oc apply -f pvc.yaml -n i27-storage-demo
```

---

## PVC with Custom StorageClass

To use the `faster` SSD-backed StorageClass, specify it explicitly:

**`pvc.yaml` — pvc-faster**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-faster
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: faster
```

Apply:

```bash
oc apply -f pvc.yaml -n i27-storage-demo
```

---

## Full `pvc.yaml`

Both PVCs can be kept in a single file separated by `---`:

```yaml
# PVC using GKE default StorageClass (standard HDD)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-standard
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
---
# PVC using custom faster StorageClass (SSD)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-faster
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: faster
```

---

## Verify

```bash
oc get pvc -n i27-storage-demo
```

Expected output:

```
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-standard   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   30Gi       RWO            standard       30s
pvc-faster     Bound    pvc-yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy   20Gi       RWO            faster         30s
```

Both PVCs should reach `Bound` status. If a PVC stays in `Pending`, the StorageClass is likely missing or the provisioner is not running.

Verify the auto-created PVs:

```bash
oc get pv
```

Check details of a specific PVC:

```bash
oc describe pvc pvc-faster -n i27-storage-demo
oc describe pvc pvc-standard -n i27-storage-demo
```

---

## PVC Status Reference

| Status | Meaning |
|--------|---------|
| Pending | Waiting for a PV to be provisioned or bound |
| Bound | Successfully bound to a PV, ready for use |
| Lost | The bound PV was deleted; data may be lost |
