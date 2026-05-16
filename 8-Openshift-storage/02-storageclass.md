# OpenShift Storage — StorageClass

## What is a StorageClass?

A StorageClass tells the cluster **how to provision storage dynamically**. It defines:

- Which **provisioner** to use (the storage backend)
- What **parameters** to pass (disk type, tier, network)
- The **reclaim policy** when the PVC is deleted
- Whether **volume expansion** is allowed

When a PVC references a StorageClass, the provisioner automatically creates a PV and binds it. No manual PV creation is needed.

---

## Default StorageClass (GKE Standard)

GKE clusters come with a default StorageClass pre-installed. When a PVC does not specify a `storageClassName`, this default is used automatically.

Verify the default StorageClass:

```bash
kubectl get sc
```

The default SC is marked with the annotation:

```
storageclass.kubernetes.io/is-default-class: "true"
```

The GKE default StorageClass uses `kubernetes.io/gce-pd` as the provisioner and provisions standard HDD-backed persistent disks.

---

## Custom StorageClass — `faster`

For workloads that need higher disk performance, a custom StorageClass backed by SSD (`pd-ssd`) can be created.

**`storageclass-faster.yaml`**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: faster
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

| Field | Value | Meaning |
|-------|-------|---------|
| `name` | `faster` | Name used in PVC's `storageClassName` field |
| `provisioner` | `kubernetes.io/gce-pd` | GCP Persistent Disk provisioner |
| `parameters.type` | `pd-ssd` | Provisions SSD-backed disk instead of HDD |

Apply:

```bash
oc apply -f storageclass-faster.yaml
```

---

## Verify

```bash
oc get sc
```

Expected output:

```
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION
faster               kubernetes.io/gce-pd    Delete          Immediate           false
standard (default)   kubernetes.io/gce-pd    Delete          Immediate           true
```

Note that the `faster` StorageClass has `ALLOWVOLUMEEXPANSION` as `false` by default. This becomes important in the volume expansion lab.

---

## Key Points

- A cluster can have multiple StorageClasses
- Only one can be marked as default
- PVCs that omit `storageClassName` use the default SC
- PVCs that specify `storageClassName: faster` use the custom SC
- StorageClass is a cluster-scoped resource (not namespaced)
