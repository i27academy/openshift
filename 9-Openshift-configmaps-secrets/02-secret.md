# OpenShift – Secret

## What is a Secret?

A Secret stores **sensitive data** such as passwords, API tokens, and TLS certificates. Like a ConfigMap, it decouples configuration from the image — but unlike a ConfigMap, the values are **base64 encoded** and access is restricted by RBAC.

---

## ConfigMap vs Secret

| | ConfigMap | Secret |
|--|-----------|--------|
| Data type | Non-sensitive config | Sensitive credentials |
| Storage | Plain text | Base64 encoded |
| Use case | DB host, port, feature flags | Passwords, tokens, certs |
| Access control | Standard RBAC | Stricter RBAC |
| Visible in `describe` | Yes | No (values hidden) |

> **Note:** Base64 is encoding, not encryption. Secrets should be protected using RBAC and, in production, encrypted at rest using etcd encryption or an external secrets manager.

---

## Secret Types

| Type | Use case |
|------|----------|
| `Opaque` | Generic key-value secrets (default) |
| `kubernetes.io/dockerconfigjson` | Image pull credentials for private registries |
| `kubernetes.io/tls` | TLS certificate and private key |

---

## Project Setup

```bash
oc project i27-config-demo
```

Apply SCC (essential for Nginx):

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-config-demo
```

---

## Creating a Secret

### From Literal Values

```bash
oc create secret generic db-secret \
  --from-literal=DB_USER=i27admin \
  --from-literal=DB_PASSWORD=i27@secure#pass \
  --from-literal=DB_ROOT_PASSWORD=i27@root#pass
```

OpenShift automatically base64 encodes the values when storing them.

---

### From a File

```bash
# db-credentials.txt
DB_USER=i27admin
DB_PASSWORD=i27@secure#pass
```

```bash
oc create secret generic db-secret --from-file=db-credentials.txt
```

---

### From YAML with Base64 Encoding

When creating a Secret via YAML, values must be base64 encoded manually.

**Encode a value:**

```bash
echo -n 'i27admin' | base64
# Output: aTI3YWRtaW4=

echo -n 'i27@secure#pass' | base64
# Output: aTI3QHNlY3VyZSNwYXNz

echo -n 'i27@root#pass' | base64
# Output: aTI3QHJvb3QjcGFzcw==
```

> Always use `echo -n` — without `-n`, echo adds a newline character which changes the encoded value.

**`db-secret.yaml`**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: i27-config-demo
type: Opaque
data:
  DB_USER: aTI3YWRtaW4=
  DB_PASSWORD: aTI3QHNlY3VyZSNwYXNz
  DB_ROOT_PASSWORD: aTI3QHJvb3QjcGFzcw==
```

Apply:

```bash
oc apply -f db-secret.yaml
```

---

### From YAML with Plain Text (stringData)

Using `stringData` lets you write plain text — OpenShift encodes it automatically on apply:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: i27-config-demo
type: Opaque
stringData:
  DB_USER: i27admin
  DB_PASSWORD: i27@secure#pass
  DB_ROOT_PASSWORD: i27@root#pass
```

`stringData` is a write-only convenience field. Once applied, the values are stored as base64 under `data`.

---

## Verify

List Secrets:

```bash
oc get secret -n i27-config-demo
```

Describe a Secret (values are hidden):

```bash
oc describe secret db-secret -n i27-config-demo
```

Output shows key names and sizes but not the actual values:

```
Name:         db-secret
Namespace:    i27-config-demo
Type:         Opaque

Data
====
DB_PASSWORD:       15 bytes
DB_ROOT_PASSWORD:  13 bytes
DB_USER:           8 bytes
```

---

## Decode a Secret Value

View the raw YAML (values are base64 encoded):

```bash
oc get secret db-secret -o yaml -n i27-config-demo
```

Decode a specific value:

```bash
oc get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' -n i27-config-demo | base64 -d
```

Extract all keys to files in a directory:

```bash
oc extract secret/db-secret -n i27-config-demo --to=./extracted/
```

Each key is saved as a separate file with the decoded value as content.
