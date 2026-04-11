
---

# 🟧 i27Academy: OpenShift Projects & Enterprise Isolation

## 1. Why Do We Need Projects? (The Business Case)
In real-world environments, multiple teams work on the same cluster:
* **Development Team**
* **Testing Team**
* **Production Team**
* **Data Team**
* **Platform Team**

Each team needs:
* Their own **applications**
* Their own **resources** (CPU/RAM)
* Their own **access control** (RBAC)

**Without separation:**
* ❌ Developers may accidentally delete production apps.
* ❌ The Test team may modify production databases.
* ❌ Resource "noisy neighbors" can crash the entire cluster.
* ❌ Everything becomes difficult to manage and audit.

**The Solution:** We need **Logical Isolation**. This is where **i27 Projects** come into the picture.

---

## 2. What is an  Project?
A Project in OpenShift is a logical isolation boundary. While Kubernetes provides "Namespaces," OpenShift enhances them into "Projects."



$$Project = Namespace + RBAC + Security\ (SCC) + UI\ Integration$$

**Everything at i27Academy runs inside a project:**
* **Workloads:** Pods, ReplicaSets, Deployments
* **Networking:** Services, Routes
* **Config:** ConfigMaps, Secrets

---

## 3. i27 Real-World Example
```text
Project: i27-dev
    └── i27-app (v2.0-beta)
    └── i27-db (Dev Instance)

Project: i27-test
    └── i27-app (v1.9-rc)
    └── i27-db (Test Instance)

Project: i27-prod
    └── i27-app (v1.8-stable)
    └── i27-db (Production Instance)
```
Each project is a walled garden. Even if they share the same physical GCP hardware, they are logically invisible to one another.

---

## 4. OpenShift Enterprise Features
* **Built-in RBAC:** Control access per team (e.g., `oc adm policy add-role-to-user edit user1 -n i27-dev`).
* **Security Context Constraints (SCC):** Makes OpenShift **secure by default** by preventing unauthorized root access.
* **UI Visibility:** The OpenShift Console allows switching projects effortlessly and viewing health metrics per environment.
* **Built-in Registry:** Each project gets an internal registry and **ImageStreams** to manage container images.
* **Project Metadata:** Supports `description` and `display-name` for better organizational clarity.

---

## 5. 📘 OC Commands Cheat Sheet

### 🔹 Login & Context
```bash
# Login to i27 Cluster
oc login --token=************ --server=https://api.i27-ocp-cluster.i27openshift.com:6443

# Check current project context
oc project
```

### 🔹 Managing Projects
```bash
# List all i27 projects
oc get projects

# Create basic projects
oc new-project i27-dev-project

oc new-project i27-demo-project \
  --display-name="i27 Demo Project" \
  --description="Created for demo purposes"

# Switch Project Context
oc project i27-demo-project

# Delete Project
oc delete project i27-demo-project
```

### 🔹 Inspection & Permissions
```bash
# Describe project details
oc describe project i27-dev-project

# View current user permissions
oc auth can-i create pods
oc auth can-i create deployments

# View who is assigned to the project
oc get rolebindings -n i27-dev-project
```

---

## 6. 📄 Declarative Management (YAML)
For production consistency, always define your **i27 Projects** using YAML.

**`i27-project.yaml`**
```yaml
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  name: dev-project
  labels:
    environment: dev
    team: platform
  annotations:
    openshift.io/description: "Development project for OpenShift training"
    openshift.io/display-name: "Development Project"
```
**Apply the file:**
```bash
oc apply -f i27-project.yaml
```

---

