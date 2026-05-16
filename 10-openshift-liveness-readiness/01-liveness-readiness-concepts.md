# OpenShift – Liveness and Readiness Probes — Concepts

## Why Probes?

Kubernetes and OpenShift can detect when a container **process crashes** and restart it automatically. But process-level detection is not enough — an app can be running but:

- Stuck in a deadlock
- Unable to serve requests due to a dependency not being ready
- Still initializing and not ready to accept traffic yet

Probes solve this by letting you define **custom health checks** that OpenShift runs against your container at regular intervals.

---

## Liveness Probe

**Question it answers:** Is the app alive and functioning?

If the liveness probe fails, OpenShift considers the container unhealthy and **restarts it**.

Use liveness probes to recover from situations where the app is running but stuck — for example, a deadlock that the app cannot recover from on its own.

```
Liveness fails → Container restarted → Fresh start
```

---

## Readiness Probe

**Question it answers:** Is the app ready to receive traffic?

If the readiness probe fails, OpenShift **removes the pod from the Service endpoints**. Traffic stops being sent to that pod. The pod is not restarted — it just stops receiving requests until it becomes ready again.

Use readiness probes to:
- Hold traffic back during slow startup
- Stop sending traffic to a pod when a dependency is temporarily unavailable

```
Readiness fails → Pod removed from Service endpoints → No traffic sent
Readiness passes → Pod added back to Service endpoints → Traffic resumes
```

---

## Liveness vs Readiness

| | Liveness | Readiness |
|--|----------|-----------|
| Question | Is the app alive? | Is the app ready for traffic? |
| On failure | Pod is restarted | Pod removed from endpoints |
| On recovery | Automatic (restart) | Automatic (re-added to endpoints) |
| RESTARTS increments? | Yes | No |
| Use case | Deadlocks, hung processes | Slow startup, dependency unavailable |

Both probes can run simultaneously on the same container and serve different purposes.

---

## Probe Types

### httpGet

Makes an HTTP GET request to a specified path and port. A response code between `200` and `399` is considered healthy.

```yaml
livenessProbe:
  httpGet:
    path: /live
    port: 3000
```

Best for web applications and APIs that expose health endpoints.

---

### tcpSocket

Attempts to open a TCP connection to the specified port. If the connection succeeds, the probe passes.

```yaml
livenessProbe:
  tcpSocket:
    port: 3000
```

Best for apps that don't expose HTTP endpoints (databases, message brokers).

---

### exec

Runs a command inside the container. If the command exits with code `0`, the probe passes.

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
```

Best for custom health logic that can be expressed as a shell command.

---

## Key Probe Fields

| Field | Meaning | Default |
|-------|---------|---------|
| `initialDelaySeconds` | Seconds to wait after container starts before running the first probe | 0 |
| `periodSeconds` | How often to run the probe | 10 |
| `timeoutSeconds` | Seconds to wait for a probe response before marking it failed | 1 |
| `failureThreshold` | How many consecutive failures before taking action | 3 |
| `successThreshold` | How many consecutive successes before marking healthy (readiness only) | 1 |

---

## Demo App — `i27devopsb8/node:livereadv1`

All labs in this series use `i27devopsb8/node:livereadv1`. This is a Node.js application that exposes:

| Endpoint | Port | Purpose |
|----------|------|---------|
| `/live` | 3000 | Liveness check — returns 200 when healthy |
| `/ready` | 3000 | Readiness check — returns 200 when ready |
| `/` | 3000 | Main application response |

Failure scenarios are simulated by pointing probes at wrong paths (`/wrong-live`, `/wrong-ready`) which return 404, causing the probes to fail intentionally.

---

## Project Setup

```bash
oc new-project i27-probes-demo
oc project i27-probes-demo
```

Apply SCC (required for this image):

```bash
oc adm policy add-scc-to-user anyuid -z default -n i27-probes-demo
```
