
-----

# OpenShift Container Platform Deployment Guide: GCP IPI

This guide provides a comprehensive, step-by-step workflow to provision an OpenShift cluster on Google Cloud using **Installer Provisioned Infrastructure (IPI)**.

-----

## Phase 1: Control Node & User Setup

**Goal:** Establish a persistent environment, elevate privileges, and verify cloud authentication.

### 1\. Provision the Installer VM

In your GCP Console, create a VM with these specifications:

  * **Name:** `ocp-installer-node`
  * **Machine Type:** `e2-medium` (2 vCPU, 4GB RAM)
  * **OS:** Ubuntu 22.04 LTS
  * **Disk:** 20GB Standard Persistent Disk

### 2\. Configure the User (`siva`)

SSH into your VM and run these commands to create a dedicated user and configure SSH access.

```bash
# 1. Create the user 'siva'
sudo adduser siva

# 2. Grant sudo privileges
sudo usermod -aG sudo siva

# 3. Configure SSH for password-based access
sudo sed -i 's/^Include \/etc\/ssh\/sshd_config.d\/\*\.conf/#&/' /etc/ssh/sshd_config
sudo sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config

# 4. Restart the SSH service
sudo systemctl restart ssh
```

### 3\. Connect, Elevate, and Initialize GCP

1.  **Login to the VM:** From your local terminal:
    `ssh siva@<VM_EXTERNAL_IP>`
2.  **Elevate to Root & Create Workspace:**
    ```bash
    sudo -i
    mkdir -p /root/ocp
    cd /root/ocp
    ```
3.  **Authenticate GCP:** `gcloud auth login`
4.  **Set Active Project:**
    ```bash
    # Set your project ID (Replace with your actual ID)
    export GCP_PROJECT_ID=<ENTER_YOUR_PROJECT_ID_HERE>
    gcloud config set project $GCP_PROJECT_ID
    ```
5.  **Verify Authentication:** `gcloud compute instances list`

-----

## 🌐 Phase 2: Red Hat Account & Pull Secret

1.  **Register:** Create an account at [**console.redhat.com**](https://www.google.com/search?q=https://console.redhat.com/).
2.  **Navigate:** **Red Hat OpenShift** \> **Create Cluster** \> **Google Cloud Platform (GCP)**.
3.  **Method:** Select **Installer-provisioned infrastructure (IPI)**.
4.  **Pull Secret:** Click **Download pull secret** or **Copy pull secret**. Keep this string ready.

-----

## ☁️ Phase 3: GCP Infrastructure & Identity Setup

### 1\. Enable Required Google APIs

```bash
gcloud services enable compute.googleapis.com cloudapis.googleapis.com \
    cloudresourcemanager.googleapis.com dns.googleapis.com \
    iamcredentials.googleapis.com iam.googleapis.com \
    servicemanagement.googleapis.com serviceusage.googleapis.com \
    storage-api.googleapis.com storage-component.googleapis.com \
    deploymentmanager.googleapis.com file.googleapis.com
```

### 2\. DNS Configuration

```bash
export GCP_DOMAIN=i27openshift.com
gcloud dns managed-zones create i27openshift-dns --dns-name=$GCP_DOMAIN. --description="DNS for Openshift"

# Retrieve Name Servers for your Domain Registrar
gcloud dns managed-zones describe i27openshift-dns
```

> **Action:** Update these Name Servers in your registrar.
> **Verify propagation:** `nslookup -q=ns $GCP_DOMAIN`

### 3\. Create Installer Service Account

```bash
export GCP_SA=i27-gcp-sa-ocp
gcloud iam service-accounts create $GCP_SA --display-name="i27 Openshift cluster sa"

gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
    --member="serviceAccount:$GCP_SA@$GCP_PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/owner"

# Generate the JSON key file using the absolute path
gcloud iam service-accounts keys create /root/ocp/i27-gcp-sa-ocp-key.json \
    --iam-account=$GCP_SA@$GCP_PROJECT_ID.iam.gserviceaccount.com
```

-----

## 📦 Phase 4: Downloading & Preparing Binaries

```bash
cd /root/ocp
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-install-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz

tar -xvzf openshift-install-linux.tar.gz
tar -xvzf openshift-client-linux.tar.gz

# Copy the client binaries to the system path (do not move)
sudo cp oc kubectl /usr/local/bin/

# Generate a new Ed25519 SSH key pair without a passphrase and save it to the default directory
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_rsa

# Start the SSH agent in the background to manage and store your private keys
eval "$(ssh-agent -s)"

# Add the newly generated private key to the SSH agent for automated authentication
ssh-add ~/.ssh/id_rsa
```

-----

## 🛠️ Phase 5: Credentials & Config Generation

```bash
export GOOGLE_APPLICATION_CREDENTIALS=/root/ocp/i27-gcp-sa-ocp-key.json
export OPENSHIFT_INSTALL_GCP_CREDENTIALS_MODE=manual

mkdir -p /root/ocp/i27-cluster
# Generate the cluster configuration
./openshift-install create install-config --dir=/root/ocp/i27-cluster
```

### Verify, Backup & Customize Configuration

Before deploying the cluster, verify the file and prepare your custom configuration.

1.  **Verify & Backup:**

    ```bash
    # Verify file exists
    ls /root/ocp/i27-cluster/install-config.yaml

    # Create a backup of the original configuration
    cp /root/ocp/i27-cluster/install-config.yaml /root/ocp/i27-cluster/install-config-ORIGINAL.yaml
    ```

2.  **Customize for Deployment:**
    You must now modify the `install-config.yaml` to meet your production requirements (e.g., node counts, machine types). Use the following link as a template:

    > **Reference Link:** [i27Academy GitHub - install-config.yaml](https://github.com/i27academy/openshift/blob/main/Cluster-Creation/install-config.yaml)

    ⚠️ **CRITICAL:** Do **not** directly copy and paste the GitHub file. It is provided strictly as a reference. You must modify your own local file based on this template to ensure specific fields like `pullSecret`, `sshKey`, and `projectID` remain correct for your environment.

-----

## 🚀 Phase 6: Launching & Verifying

### 1\. Trigger Cluster Creation (Takes 30-45 mins)

```bash
./openshift-install create cluster --dir=/root/ocp/i27-cluster --log-level debug
```

### 2\. Accessing the Cluster

```bash
export KUBECONFIG=/root/ocp/i27-cluster/auth/kubeconfig
# Get login info
cat /root/ocp/i27-cluster/auth/kubeadmin-password
oc get route console -n openshift-console
```

### ⚠️ Cleanup

```bash
./openshift-install destroy cluster --dir=/root/ocp/i27-cluster --log-level debug
```
