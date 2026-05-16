# OpenShift Storage — GCP Filestore (RWX Shared Storage)

## Why RWX?

GCP Persistent Disks support **RWO (ReadWriteOnce)** — only one node can mount the volume at a time. This means a deployment with multiple replicas spread across different nodes cannot share a single PD-backed volume.

For use cases that require multiple pods on different nodes to read and write the same storage simultaneously, **RWX (ReadWriteMany)** is needed.

GCP Filestore provides NFS-backed storage that supports RWX, making it suitable for shared file access across pods.

---

## Architecture

```
Pod 1 ──┐
         ├──▶ PVC ──▶ StorageClass (sc-filestore-rwx) ──▶ CSI Driver ──▶ GCP Filestore (NFS)
Pod 2 ──┘
```

---

## Prerequisites

- OpenShift cluster running on GCP
- `oc` CLI with cluster-admin access
- GCP project access

---

## Step 1 — Enable Filestore API

```bash
gcloud services enable file.googleapis.com --project <PROJECT_ID>
```

---

## Step 2 — Install the Filestore CSI Operator

In the OpenShift web console:

```
Operators → OperatorHub → Search: "Google Cloud Filestore CSI"
```

Install into namespace:

```
openshift-cluster-csi-drivers
```

### What is an Operator?

An Operator is a Kubernetes component that automates deployment, configuration, and lifecycle management of complex applications. In this case, the Filestore CSI Operator:

- Deploys the CSI driver pods
- Connects OpenShift to GCP Filestore
- Enables dynamic provisioning of Filestore instances
- Handles recovery and scaling

---

## Step 3 — Enable the CSI Driver

**`filestore-csidriver.yaml`**

```yaml
apiVersion: operator.openshift.io/v1
kind: ClusterCSIDriver
metadata:
  name: filestore.csi.storage.gke.io
spec:
  managementState: Managed
```

```bash
oc apply -f filestore-csidriver.yaml
```

Verify the CSI driver pods are running:

```bash
oc get pods -n openshift-cluster-csi-drivers | grep filestore
oc get csidriver
```

Expected in `oc get csidriver` output:

```
filestore.csi.storage.gke.io
```

---

## Step 4 — Create the StorageClass

**`storageclass-filestore-rwx.yaml`**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-filestore-rwx
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: basic_hdd
  network: default
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

```bash
oc apply -f storageclass-filestore-rwx.yaml
```

---

## Step 5 — Create the PVC

**`pvc-filestore-rwx.yaml`**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-filestore-rwx
  namespace: i27-storage-demo
spec:
  storageClassName: sc-filestore-rwx
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Ti
```

```bash
oc apply -f pvc-filestore-rwx.yaml
```

The PVC will stay in `Pending` until a pod is scheduled (because `volumeBindingMode: WaitForFirstConsumer`). This is expected.

---

## Step 6 — Deploy the Application

Two replicas both write to the same shared volume to demonstrate RWX behavior.

**`deployment-filestore-rwx.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-filestore-rwx
  namespace: i27-storage-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: deploy-filestore-rwx
  template:
    metadata:
      labels:
        app: deploy-filestore-rwx
    spec:
      containers:
      - name: busybox
        image: busybox
        command:
          - sh
          - -c
          - |
            while true; do
              echo "$(hostname) - i27Academy" >> /data/output.txt;
              sleep 5;
            done
        volumeMounts:
        - name: filestore-volume
          mountPath: /data
      volumes:
      - name: filestore-volume
        persistentVolumeClaim:
          claimName: pvc-filestore-rwx
```

```bash
oc apply -f deployment-filestore-rwx.yaml
```

---

## Step 7 — Verify

Watch the PVC bind once the pods are scheduled:

```bash
oc get pvc -n i27-storage-demo -w
```

Check pods are running:

```bash
oc get pods -n i27-storage-demo
```

Check the PV was auto-created:

```bash
oc get pv
```

---

## Step 8 — Validate RWX (Shared Storage)

Get both pod names:

```bash
oc get pods -n i27-storage-demo
```

Read the shared file from pod 1:

```bash
oc exec -it <pod-1-name> -n i27-storage-demo -- cat /data/output.txt
```

Read the same file from pod 2:

```bash
oc exec -it <pod-2-name> -n i27-storage-demo -- cat /data/output.txt
```

Both pods will show entries from each other, confirming the shared volume is working:

```
deploy-filestore-rwx-xxxxx - i27Academy
deploy-filestore-rwx-yyyyy - i27Academy
deploy-filestore-rwx-xxxxx - i27Academy
deploy-filestore-rwx-yyyyy - i27Academy
```

---

## Step 9 — Verify in GCP Console

```
Google Cloud Console → Filestore → Instances
```

You will see a Filestore instance automatically created by the CSI driver when the PVC was bound.

---

## Cleanup

> **Important:** Filestore instances incur cost. Always clean up after the lab.

Delete the deployment first:

```bash
oc delete deployment deploy-filestore-rwx -n i27-storage-demo
```

Delete the PVC (this triggers deletion of the Filestore instance due to `reclaimPolicy: Delete`):

```bash
oc delete pvc pvc-filestore-rwx -n i27-storage-demo
```

Verify the PV is also gone:

```bash
oc get pv
```

Delete the StorageClass:

```bash
oc delete sc sc-filestore-rwx
```

Verify in GCP Console that the Filestore instance has been removed:

```
Google Cloud Console → Filestore → Instances
```
