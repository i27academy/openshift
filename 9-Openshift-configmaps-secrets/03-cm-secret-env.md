# OpenShift – ConfigMap and Secret as Environment Variables

## Overview

Both ConfigMaps and Secrets can be injected into pods as environment variables. There are two patterns:

| Pattern | Field | Behaviour |
|---------|-------|-----------|
| Single key | `configMapKeyRef` / `secretKeyRef` | Inject one specific key as a named env var |
| All keys | `configMapRef` / `secretRef` (via `envFrom`) | Inject every key in the CM or Secret as env vars |

---

## Prerequisites

ConfigMap `db-config` and Secret `db-secret` must exist in `i27-config-demo`. See `01-configmap.md` and `02-secret.md` for creation steps.

Apply SCC (essential for Nginx):

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-config-demo
```

---

## Pattern 1 — Single Key from ConfigMap (`configMapKeyRef`)

Injects one specific key from the ConfigMap as an environment variable with a custom name.

```yaml
env:
- name: DATABASE_HOST        # env var name inside the container
  valueFrom:
    configMapKeyRef:
      name: db-config        # ConfigMap name
      key: DB_HOST           # key within the ConfigMap
- name: DATABASE_PORT
  valueFrom:
    configMapKeyRef:
      name: db-config
      key: DB_PORT
```

Use this when you need to rename a key or only inject specific values.

---

## Pattern 2 — All Keys from ConfigMap (`configMapRef`)

Injects every key from the ConfigMap as environment variables. Key names become the env var names directly.

```yaml
envFrom:
- configMapRef:
    name: db-config          # all keys from this ConfigMap are injected
```

Use this when you want all values from a ConfigMap without listing them individually.

---

## Pattern 3 — Single Key from Secret (`secretKeyRef`)

Injects one specific key from the Secret as an environment variable.

```yaml
env:
- name: DATABASE_USER
  valueFrom:
    secretKeyRef:
      name: db-secret        # Secret name
      key: DB_USER           # key within the Secret
- name: DATABASE_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: DB_PASSWORD
```

---

## Pattern 4 — All Keys from Secret (`secretRef`)

Injects every key from the Secret as environment variables.

```yaml
envFrom:
- secretRef:
    name: db-secret          # all keys from this Secret are injected
```

---

## Full Deployment YAML — All 4 Patterns

**`deployment-env.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-env-demo
  namespace: i27-config-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: env-demo
  template:
    metadata:
      labels:
        app: env-demo
    spec:
      containers:
      - name: nginx
        image: devopswithcloudhub/nginx:blue
        ports:
        - containerPort: 80

        # Pattern 1: Single key from ConfigMap
        env:
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_HOST
        - name: DATABASE_PORT
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_PORT

        # Pattern 3: Single key from Secret
        - name: DATABASE_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_USER
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD

        # Pattern 2 + 4: All keys from ConfigMap and Secret
        envFrom:
        - configMapRef:
            name: db-config
        - secretRef:
            name: db-secret
```

Apply:

```bash
oc apply -f deployment-env.yaml
```

---

## Verify

Check pod is running:

```bash
oc get pods -n i27-config-demo
```

Exec into the pod and print all environment variables:

```bash
oc exec -it <pod-name> -n i27-config-demo -- printenv
```

Filter for specific variables:

```bash
# From ConfigMap (single key pattern)
oc exec -it <pod-name> -n i27-config-demo -- printenv DATABASE_HOST
oc exec -it <pod-name> -n i27-config-demo -- printenv DATABASE_PORT

# From ConfigMap (all keys pattern)
oc exec -it <pod-name> -n i27-config-demo -- printenv DB_NAME

# From Secret (single key pattern)
oc exec -it <pod-name> -n i27-config-demo -- printenv DATABASE_USER
oc exec -it <pod-name> -n i27-config-demo -- printenv DATABASE_PASSWORD

# From Secret (all keys pattern)
oc exec -it <pod-name> -n i27-config-demo -- printenv DB_ROOT_PASSWORD
```

---

## Important Notes

- If a referenced ConfigMap or Secret **does not exist**, the pod will fail to start with `CreateContainerConfigError`
- `envFrom` injects all keys — if two sources have the same key name, the last one wins
- Env var changes require a **pod restart** to take effect — updating the ConfigMap or Secret alone is not enough
- Use `optional: true` on `configMapKeyRef` or `secretKeyRef` if the key may not always exist:

```yaml
env:
- name: DATABASE_HOST
  valueFrom:
    configMapKeyRef:
      name: db-config
      key: DB_HOST
      optional: true
```
