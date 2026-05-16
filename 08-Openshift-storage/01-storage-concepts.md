# OpenShift Storage — Concepts and Overview

## Why Persistent Storage?

Pods are ephemeral. When a pod is deleted or restarted, everything written inside the container's filesystem is lost. For applications that need to retain data — databases, file uploads, logs — this is a problem.

Persistent storage solves this by attaching an external volume to the pod. The volume exists independently of the pod lifecycle. If the pod is deleted and recreated, the new pod can mount the same volume and access the same data.

---

## Key Objects

### PersistentVolume (PV)

A PV is a piece of storage provisioned in the cluster. It represents the actual storage resource — a GCP disk, an NFS share, etc.

- Created manually (static) or automatically (dynamic)
- Exists independently of any pod or PVC
- Has a capacity, access mode, and reclaim policy

### PersistentVolumeClaim (PVC)

A PVC is a request for storage made by a user or deployment. It specifies how much storage is needed and what access mode is required.

- The cluster finds or creates a matching PV and **binds** it to the PVC
- Pods reference the PVC, not the PV directly

### StorageClass (SC)

A StorageClass defines **how** storage is provisioned dynamically. It specifies the provisioner (the storage backend) and parameters like disk type.

- When a PVC references a StorageClass, a PV is created automatically
- This is called **dynamic provisioning**

---

## Dynamic Provisioning Flow

```
PVC created
    ↓
StorageClass referenced
    ↓
Provisioner creates PV automatically (e.g. GCP disk)
    ↓
PVC status changes: Pending → Bound
    ↓
Pod mounts the PVC
```

Without a StorageClass, you would need to manually create PVs before any PVC could bind — this is static provisioning and is rarely used in cloud environments.

---

## Access Modes

Access modes define how a volume can be mounted across nodes.

| Mode | Short | Meaning |
|------|-------|---------|
| ReadWriteOnce | RWO | Mounted by a single node (read/write) |
| ReadWriteMany | RWX | Mounted by multiple nodes simultaneously (read/write) |
| ReadOnlyMany | ROX | Mounted by multiple nodes (read only) |

GCP Persistent Disks support **RWO only**. For RWX (shared access across multiple pods on different nodes), GCP Filestore is used.

---

## Reclaim Policies

Reclaim policy controls what happens to the PV when the PVC is deleted.

| Policy | Behavior |
|--------|----------|
| Delete | PV and the underlying storage are deleted automatically |
| Retain | PV remains, data is preserved, manual cleanup required |

In cloud lab environments, `Delete` is the default and is appropriate. In production, `Retain` is used when data must be preserved.

---

## Lab Namespace

All storage labs in this series use:

```bash
oc new-project i27-storage-demo
oc project i27-storage-demo
```
