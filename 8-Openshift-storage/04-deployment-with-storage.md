# OpenShift Storage — Deployment with PVC

## Overview

Once PVCs are bound, they can be mounted into a Deployment. The pod spec references the PVC under `volumes`, and the container references the volume under `volumeMounts`.

This lab runs two deployments — one using the standard PVC and one using the faster SSD PVC — to demonstrate that storage works independently per claim.

---

## Prerequisites

Both PVCs must be in `Bound` status before applying deployments. Verify:

```bash
oc get pvc -n i27-storage-demo
```

---

## Deployment YAML

**`deployment-with-pvc.yaml`**

```yaml
# Deployment using standard storage (default SC)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-standard-storage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox-standard
  template:
    metadata:
      labels:
        app: busybox-standard
    spec:
      volumes:
      - name: standard-volume
        persistentVolumeClaim:
          claimName: pvc-standard
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]
        volumeMounts:
        - name: standard-volume
          mountPath: "/mnt/standard-data"
---
# Deployment using faster storage (SSD SC)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-faster-storage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox-faster
  template:
    metadata:
      labels:
        app: busybox-faster
    spec:
      volumes:
      - name: faster-volume
        persistentVolumeClaim:
          claimName: pvc-faster
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]
        volumeMounts:
        - name: faster-volume
          mountPath: "/mnt/faster-data"
```

Apply:

```bash
oc apply -f deployment-with-pvc.yaml -n i27-storage-demo
```

---

## Verify Pods

```bash
oc get pods -n i27-storage-demo
```

Both pods should reach `Running` status. If a pod stays in `ContainerCreating`, the PVC is likely not yet bound.

---

## Test Storage is Working

Get pod names:

```bash
oc get pods -n i27-storage-demo
```

Write data into the standard storage pod:

```bash
oc exec -it <deploy-standard-storage-pod> -n i27-storage-demo -- \
  sh -c "echo 'standard storage test' > /mnt/standard-data/test.txt"
```

Read it back:

```bash
oc exec -it <deploy-standard-storage-pod> -n i27-storage-demo -- \
  cat /mnt/standard-data/test.txt
```

Write data into the faster storage pod:

```bash
oc exec -it <deploy-faster-storage-pod> -n i27-storage-demo -- \
  sh -c "echo 'faster storage test' > /mnt/faster-data/test.txt"
```

Read it back:

```bash
oc exec -it <deploy-faster-storage-pod> -n i27-storage-demo -- \
  cat /mnt/faster-data/test.txt
```

---

## Test Data Persistence

Delete the pod manually (the deployment will recreate it):

```bash
oc delete pod <deploy-standard-storage-pod> -n i27-storage-demo
```

Wait for the new pod to come up:

```bash
oc get pods -n i27-storage-demo -w
```

Read the file from the new pod:

```bash
oc exec -it <new-pod-name> -n i27-storage-demo -- \
  cat /mnt/standard-data/test.txt
```

The data written before the pod was deleted is still present. The volume persists independently of the pod lifecycle.

---

## How Volumes Are Wired

```
Deployment spec
    └── volumes
            └── name: standard-volume
                    claimName: pvc-standard     ← references the PVC
    └── containers
            └── volumeMounts
                    └── name: standard-volume   ← must match volume name above
                        mountPath: /mnt/standard-data
```

The `name` field in `volumes` and `volumeMounts` must match exactly. The `mountPath` is where the storage appears inside the container.
