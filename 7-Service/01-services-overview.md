# OpenShift Services — Overview

## What is a Service?

A **Service** in OpenShift is an object that provides stable network access to a group of Pods.

A Service gives:

- A stable name
- A stable IP (ClusterIP)
- Load balancing across all matching Pods

---

## Why is a Service Needed?

Pods are **ephemeral** — they can be deleted and recreated at any time, and their IPs change when that happens. Connecting directly to a Pod IP is unreliable.

A Service sits in front of Pods and provides a stable endpoint. Even if the Pods behind it are replaced, the Service address stays the same.

**Example:** Three Pods running the same application:

```
Pod 1 → 10.128.0.10
Pod 2 → 10.128.0.11
Pod 3 → 10.128.0.12
```

Instead of tracking Pod IPs, other applications connect to the Service name:

```
nginx-service
```

If a Pod is replaced and gets a new IP, the Service automatically updates. The caller is unaffected.

---

## Service Types

### ClusterIP (Default)

- Accessible only inside the cluster
- Used for internal Pod-to-Pod communication
- Not reachable from a browser or external network

### NodePort

- Exposes the Service on a port of each Node
- Accessible using `NodeIP:NodePort`
- Useful for testing and temporary external access

### LoadBalancer

- Provisions an external load balancer via the cloud provider (AWS, GCP, Azure)
- Gives a public IP for external access
- Requires cloud integration to be configured

---

## OpenShift-Specific: Route

In OpenShift, web applications are typically exposed using:

```
Deployment → Service → Route
```

- **Service** provides internal cluster access
- **Route** exposes the Service externally via a hostname

Route is the preferred way to expose applications in OpenShift rather than NodePort or LoadBalancer.

---

## How Services Find Pods

Services use **labels and selectors**.

The Service's `selector` field matches the labels on Pods. All Pods with a matching label become **endpoints** for that Service.

Example:

```yaml
# Pod label
labels:
  app: nginx-app

# Service selector
selector:
  app: nginx-app
```

---

## DNS in ClusterIP

When a Service is created, OpenShift automatically registers a DNS name for it.

**Full DNS format:**

```
<service-name>.<namespace>.svc.cluster.local
```

**Example:**

```
nginx-clusterip-service.i27-service-demo.svc.cluster.local
```

**Short name** (works within the same namespace):

```
nginx-clusterip-service
```

In real applications, always use DNS names rather than Pod IPs or hardcoded ClusterIPs:

```
frontend  →  backend-service
backend   →  mysql-service
payment   →  auth-service
```

---

## Lab Setup

All labs in this series use the project `i27-service-demo` and the image `devopswithcloudhub/nginx:blue`.

### Create the project

```bash
oc new-project i27-service-demo
oc project i27-service-demo
```

### Apply SCC (required for this image)

Some container images are built to run as root. OpenShift blocks this by default. For lab purposes, grant `anyuid` SCC to the default service account:

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-service-demo
```

| Flag | Meaning |
|------|---------|
| `anyuid` | Allow the container to run with any user ID |
| `-z default` | Apply to the `default` ServiceAccount |
| `-n i27-service-demo` | Scoped to this project only |

> **Note:** This is a lab convenience setting, not a production practice. SCC hardening is covered separately.

### Deploy the application

```bash
oc create deployment nginx-app --image=devopswithcloudhub/nginx:blue
oc get deploy
oc get pods -o wide
oc get pods --show-labels
```

The deployment creates Pods with the label `app=nginx-app`, which all Services in the following labs use as their selector.
