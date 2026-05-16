# OpenShift – ConfigMap

## What is a ConfigMap?

A ConfigMap stores **non-sensitive configuration data** as key-value pairs. It decouples configuration from the container image, so the same image can be used across different environments (dev, staging, production) by simply swapping the ConfigMap.

Without a ConfigMap, configuration values like database hostnames, ports, or feature flags would need to be hardcoded into the image or passed manually — making the setup brittle and environment-specific.

---

## Project Setup

```bash
oc new-project i27-config-demo
oc project i27-config-demo
```

Apply SCC (essential for Nginx):

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-config-demo
```

---

## Creating a ConfigMap

### From Literal Values

```bash
oc create configmap db-config \
  --from-literal=DB_HOST=mysql.i27-config-demo.svc.cluster.local \
  --from-literal=DB_PORT=3306 \
  --from-literal=DB_NAME=i27academy
```

Each `--from-literal` flag adds one key-value pair to the ConfigMap.

---

### From a File

If your configuration lives in a file (e.g. `app.properties`):

```bash
# app.properties
DB_HOST=mysql.i27-config-demo.svc.cluster.local
DB_PORT=3306
DB_NAME=i27academy
```

```bash
oc create configmap db-config --from-file=app.properties
```

The filename becomes the key, and the file contents become the value.

To use a custom key name instead of the filename:

```bash
oc create configmap db-config --from-file=config=app.properties
```

---

### From a Directory

If a directory contains multiple config files:

```bash
oc create configmap db-config --from-file=./config-dir/
```

Each file in the directory becomes a separate key in the ConfigMap.

---

### From YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
  namespace: i27-config-demo
data:
  DB_HOST: mysql.i27-config-demo.svc.cluster.local
  DB_PORT: "3306"
  DB_NAME: i27academy
  app.properties: |
    DB_HOST=mysql.i27-config-demo.svc.cluster.local
    DB_PORT=3306
    DB_NAME=i27academy
```

Apply:

```bash
oc apply -f db-config.yaml
```

The `data` block supports both simple key-value pairs and multi-line file-like content using the `|` block scalar.

---

## Verify

List ConfigMaps:

```bash
oc get cm -n i27-config-demo
```

Describe a ConfigMap (shows keys and values):

```bash
oc describe cm db-config -n i27-config-demo
```

View the full YAML including all values:

```bash
oc get cm db-config -o yaml -n i27-config-demo
```

Expected output from `oc get cm`:

```
NAME        DATA   AGE
db-config   3      10s
```

---

## Key Points

- ConfigMap data is stored as **plain text** — never use it for passwords or tokens
- Values are always strings — numbers must be quoted in YAML (`"3306"`)
- ConfigMaps are **namespaced** — they must exist in the same namespace as the pod using them
- Updating a ConfigMap does not automatically restart pods that use it as environment variables — pods must be restarted manually
- Pods using ConfigMaps as **volumes** pick up changes automatically (with a short delay)
