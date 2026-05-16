# OpenShift Services — NodePort

## What is NodePort?

NodePort exposes the Service on a specific port of **every Node** in the cluster. It can be accessed externally using:

```
http://<NodeIP>:<NodePort>
```

NodePort is primarily used for testing, demos, and temporary external access. In production OpenShift environments, Routes are preferred over NodePort.

---

## Prerequisites

Project `i27-service-demo` created and `nginx-app` deployment running. See `01-services-overview.md` for setup steps.

---

## NodePort YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-service
spec:
  selector:
    app: nginx-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
  type: NodePort
```

---

## Apply

```bash
oc apply -f nodeport.yaml
```

Verify:

```bash
oc get svc nginx-nodeport-service
```

Expected output:

```
NAME                     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-nodeport-service   NodePort   172.30.55.10    <none>        80:30080/TCP   10s
```

---

## Get Node IP

```bash
oc get nodes -o wide
```

Note the `EXTERNAL-IP` or `INTERNAL-IP` of any reachable node.

---

## Access the Application

```bash
curl http://<NodeIP>:30080
```

Or open in a browser:

```
http://<NodeIP>:30080
```

---

## Important Note

NodePort may not be reachable externally if:

- A firewall is blocking the port
- The Node does not have a public IP
- Cloud security/network rules are restricting traffic

If access fails, verify firewall rules and confirm the Node IP is reachable from your network.
