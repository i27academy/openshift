
---

# 🔐 Securing OpenShift Console with Trusted SSL Certificate

## 📌 Problem Statement (Real-World Scenario)
After deploying an **OpenShift Container Platform** cluster on **GCP**, the Web Console is accessible but the browser displays **“Not Secure” / Certificate Warning**.

**Example:** `https://console-openshift-console.apps.i27-cluster.i27openshift.com`

This creates:
* ❌ **Trust issues** for users and stakeholders.
* ❌ **Compliance concerns** for production environments.
* ❌ **Manual overhead** (constantly bypassing browser warnings).

---

## 🧠 Root Cause & Industry Solution
By default, OpenShift uses **self-signed certificates**. While traffic is encrypted, browsers do not trust the internal CA. 

**The Solution:** Use a **public Certificate Authority (CA)** like **Let's Encrypt** to issue a **wildcard certificate** ($*.apps.i27-cluster.i27openshift.com$). This single certificate secures the Console, OAuth, and all future application routes.

---

## 🛠️ Step-by-Step Implementation

### 1️⃣ Install Certbot
Run this on your installer VM or bastion host:
```bash
sudo apt update && sudo apt install -y certbot
```

### 2️⃣ Request Wildcard Certificate
Wildcard certificates require the **DNS-01 Challenge** to prove domain ownership.
```bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  -d "*.apps.i27-cluster.i27openshift.com" \
  -d "apps.i27-cluster.i27openshift.com"
```

### 3️⃣ Validate Domain Ownership (GCP Cloud DNS)
Certbot will pause and provide a **TXT Record** value. 

1.  **Go to GCP Console** > **Network Services** > **Cloud DNS**.
2.  **Select your zone** (`i27openshift-dns`).
3.  **Add a record:**
    * **Type:** `TXT`
    * **Name:** `_acme-challenge.apps.i27-cluster`
    * **TTL:** `300`
    * **Value:** Paste the token provided by Certbot.
4.  **Verify:** Open a new terminal and run:
    `dig TXT _acme-challenge.apps.i27-cluster.i27openshift.com`
5.  **Confirm:** Once the token appears in the `dig` output, go back to the Certbot terminal and press **Enter**.



### 4️⃣ Create TLS Secret in OpenShift
Once issued, navigate to the certificate directory and upload the keys to the cluster:
```bash
cd /etc/letsencrypt/live/apps.i27-cluster.i27openshift.com/

oc create secret tls letsencrypt-ingress \
  --cert=fullchain.pem \
  --key=privkey.pem \
  -n openshift-ingress
```

### 5️⃣ Attach Certificate to Ingress Controller
Patch the operator to switch from self-signed to your new trusted secret:
```bash
oc patch ingresscontroller default \
  -n openshift-ingress-operator \
  --type=merge \
  -p '{"spec": {"defaultCertificate": {"name": "letsencrypt-ingress"}}}'
```

---

## 🚀 Final Verification

### 6️⃣ Verify Router Pods
The Ingress Operator will automatically restart the router pods to apply the new configuration.
```bash
oc get pods -n openshift-ingress
```
> **Note:** Monitor the pods until the new instances are in **Running** status.

### 7️⃣ Browser Validation
Open your browser and navigate to your console URL.
* ✅ **Secure Lock Icon** is now visible.
* ✅ **No Certificate Warnings.**
* ✅ **Verified by Let's Encrypt.**

---

## 🏢 Real-Time Enterprise Practices

* **Environment Strategy:** Use Public CAs for Production and Internal CAs/Self-signed for Dev/QA.
* **Automation:** Let's Encrypt certificates expire every **90 days**. Use **cert-manager** in production to automate these steps.
* **Ingress Level:** Always apply certificates at the **Ingress level** to ensure all application routes inherit the trust automatically.

---

### 🧩 Summary
OpenShift is secure by default, but not "trusted" by browsers. By implementing a **Wildcard SSL via Let’s Encrypt**, you solve the trust issue across the entire platform with one clean configuration.