# OpenShift – ConfigMap and Secret as Volumes

## Overview

ConfigMaps and Secrets can be mounted as volumes inside a pod. When mounted:

- Each **key** in the ConfigMap or Secret becomes a **file**
- The **value** of that key becomes the **file content**
- The files appear at the specified `mountPath` inside the container

This is the preferred approach for:

- Config files (`app.properties`, `nginx.conf`)
- TLS certificates and private keys
- SSH keys
- API token files

Unlike environment variables, volume-mounted ConfigMaps **update automatically** inside the pod when the ConfigMap is edited (with a short sync delay). Secrets mounted as volumes also update, but with a slightly longer delay depending on the kubelet sync period.

---

## Prerequisites

ConfigMap `db-config` and Secret `db-secret` must exist in `i27-config-demo`. See `01-configmap.md` and `02-secret.md` for creation steps.

Apply SCC (essential for Nginx):

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-config-demo
```

---

## How Volume Mounting Works

```
ConfigMap: db-config
  key: DB_HOST  → value: mysql.i27-config-demo.svc.cluster.local
  key: DB_PORT  → value: 3306
  key: DB_NAME  → value: i27academy

Mounted at /etc/config/

Inside container:
  /etc/config/DB_HOST   ← file, contents: mysql.i27-config-demo.svc.cluster.local
  /etc/config/DB_PORT   ← file, contents: 3306
  /etc/config/DB_NAME   ← file, contents: i27academy
```

---

## Full Deployment YAML — CM and Secret as Volumes

**`deployment-volume.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-volume-demo
  namespace: i27-config-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: volume-demo
  template:
    metadata:
      labels:
        app: volume-demo
    spec:
      containers:
      - name: nginx
        image: devopswithcloudhub/nginx:blue
        ports:
        - containerPort: 80

        volumeMounts:
        # ConfigMap mounted as volume
        - name: config-volume
          mountPath: /etc/config          # each CM key becomes a file here

        # Secret mounted as volume
        - name: secret-volume
          mountPath: /etc/secret          # each Secret key becomes a file here
          readOnly: true                  # always mount secrets as readOnly

      volumes:
      # Reference ConfigMap
      - name: config-volume
        configMap:
          name: db-config

      # Reference Secret
      - name: secret-volume
        secret:
          secretName: db-secret
```

Apply:

```bash
oc apply -f deployment-volume.yaml
```

---

## Verify

Check pod is running:

```bash
oc get pods -n i27-config-demo
```

Exec into the pod:

```bash
oc exec -it <pod-name> -n i27-config-demo -- sh
```

Inside the pod — verify ConfigMap files:

```sh
# List all files from ConfigMap
ls /etc/config/

# Read individual keys
cat /etc/config/DB_HOST
cat /etc/config/DB_PORT
cat /etc/config/DB_NAME
```

Inside the pod — verify Secret files:

```sh
# List all files from Secret
ls /etc/secret/

# Read individual keys
cat /etc/secret/DB_USER
cat /etc/secret/DB_PASSWORD
cat /etc/secret/DB_ROOT_PASSWORD
```

Exit:

```sh
exit
```

---

## Mounting a Specific Key Only

By default, all keys are mounted as files. To mount only a specific key:

```yaml
volumes:
- name: config-volume
  configMap:
    name: db-config
    items:
    - key: DB_HOST          # only this key is mounted
      path: database-host   # mounted as this filename
```

Inside the container, only `/etc/config/database-host` will exist.

---

## Mounting a ConfigMap as a Single Config File

For use cases like `nginx.conf` or `app.properties`, the entire file content is stored as one CM key and mounted at a specific path:

ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: i27-config-demo
data:
  nginx.conf: |
    server {
      listen 80;
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
    }
```

Deployment volume mount:

```yaml
volumeMounts:
- name: nginx-config-volume
  mountPath: /etc/nginx/conf.d/nginx.conf
  subPath: nginx.conf             # mount only this key as a single file

volumes:
- name: nginx-config-volume
  configMap:
    name: nginx-config
```

`subPath` mounts a single key as a file at the exact path without affecting other files in that directory.

---

## CM vs Secret Volume Behaviour

| | ConfigMap Volume | Secret Volume |
|--|-----------------|---------------|
| File permissions | 0644 by default | 0644 by default |
| Mount as readOnly | Optional | Recommended |
| Auto-updates in pod | Yes (kubelet sync) | Yes (kubelet sync) |
| Use for | Config files, properties | Certs, keys, tokens |
