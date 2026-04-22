# MediMesh — Complete Implementation Walkthrough

## From Zero to Production: Full DevOps Implementation Guide

**Project:** MediMesh — Hospital Management Microservices Platform <br>
**Organization:** Medimesh-grp3  
**Stack:** Node.js · Docker · Kubernetes · Helm · ArgoCD · GitHub Actions · Prometheus · Grafana  
**Document:** Complete Technical Implementation Guide  

---

## 📑 Table of Contents

1. [Infrastructure Setup — Kubernetes Cluster](#1-infrastructure-setup)
2. [Node Preparation](#2-node-preparation)
3. [Cluster Initialization](#3-cluster-initialization)
4. [CNI Plugin — Flannel](#4-cni-plugin--flannel)
5. [NFS Server Setup](#5-nfs-server-setup)
6. [Writing Dockerfiles](#6-writing-dockerfiles)
7. [Kubernetes Namespaces & Helm Charts](#7-kubernetes-namespaces--helm-charts)
8. [Sealed Secrets Setup](#8-sealed-secrets-setup)
9. [Argo Rollouts Setup](#9-argo-rollouts-setup)
10. [ArgoCD Setup](#10-argocd-setup)
11. [kGateway Setup](#11-kgateway-setup)
12. [Helm Deployment](#12-helm-deployment)
13. [Prometheus & Grafana Setup](#13-prometheus--grafana-setup)
14. [SonarQube Setup](#14-sonarqube-setup)
15. [GitHub Actions CI Setup](#15-github-actions-ci-setup)
16. [GitHub Actions CD Setup](#16-github-actions-cd-setup)
17. [OIDC Setup for Vitals](#17-oidc-setup-for-vitals)
18. [Verifying the Full Pipeline](#18-verifying-the-full-pipeline)
19. [Troubleshooting Reference](#19-troubleshooting-reference)

---

## 1. Infrastructure Setup

### Overview

MediMesh runs on a self-managed Kubernetes cluster built using `kubeadm`.
The cluster has one master (control plane) node and two worker nodes. 
A separate NFS server provides persistent storage for MongoDB.

### 1.1 VM Requirements

| Node       | Role          | CPU    | RAM  | Disk  |
| ---------- | ------------- | ------ | ---- | ----- |
| master     | Control Plane | 4 vCPU | 8 GB | 30 GB |
| worker-1   | Worker Node   | 2 vCPU | 4 GB | 20 GB |
| worker-2   | Worker Node   | 2 vCPU | 4 GB | 20 GB |
| nfs-server | NFS Storage   | 2 vCPU | 2 GB | 50 GB |

> **OS:** Ubuntu 22.04 LTS on all nodes
> **Network:** All nodes must be able to reach each other

---

## 2. Node Preparation

> ⚠️ Run ALL commands in this section on
> **EVERY node** (master + worker-1 + worker-2)

### Why this is needed

Kubernetes requires specific kernel modules, network settings, and a container runtime before the cluster can be formed.
Swap must be disabled because Kubernetes memory management conflicts with Linux swap behavior.

---

### 2.1 System Update

```bash
sudo apt-get update -y
sudo apt-get upgrade -y
```

---

### 2.2 Disable Swap

Kubernetes requires swap to be disabled.
If swap is active, kubelet will refuse to start.


```bash
# Disable swap immediately
sudo swapoff -a

# Disable swap permanently (survives reboot)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is off
free -h
# Swap line should show: 0B  0B  0B
```

---

### 2.3 Load Required Kernel Modules

These modules are needed for container networking.

* `overlay` — used by containerd for layered filesystems
* `br_netfilter` — allows iptables to see bridged traffic

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

# Load modules immediately (no reboot needed)
sudo modprobe overlay
sudo modprobe br_netfilter

# Verify modules are loaded
lsmod | grep overlay
lsmod | grep br_netfilter
```

---

### 2.4 Set Kernel Parameters

These settings allow Kubernetes networking (iptables) to ee and manage bridge network traffic between pods.
```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply settings immediately
sudo sysctl --system

# Verify
sysctl net.ipv4.ip_forward
# Should output: net.ipv4.ip_forward = 1
```

---

### 2.5 Install Container Runtime (containerd)

`containerd` is the industry-standard container runtime used by Kubernetes. It replaced Docker as the default runtime in Kubernetes 1.24+.
```bash
# Install required packages
sudo apt-get install -y \
  curl \
  gnupg2 \
  software-properties-common \
  apt-transport-https \
  ca-certificates

# Add Docker GPG key (containerd is in Docker's repo)
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor \
  -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt-get update -y

# Install containerd
sudo apt-get install -y containerd.io

# Generate default configuration
containerd config default \
  | sudo tee /etc/containerd/config.toml >/dev/null 2>&1

# Enable SystemdCgroup (required for Kubernetes)
sudo sed -i \
  's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

# Restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd

# Verify containerd is running
sudo systemctl status containerd
```

---

### 2.6 Install Kubernetes Components

* `kubelet` — runs on every node, manages pods
* `kubeadm` — bootstraps the cluster
* `kubectl` — CLI to interact with the cluster

```bash
# Add Kubernetes GPG key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key \
  | sudo gpg --dearmor \
  -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo \
  'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y

# Install and pin version to avoid accidental upgrades
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable kubelet

# Verify
kubectl version --client
kubeadm version
```

---

## 3. Cluster Initialization

⚠️ Run this section ONLY on the master node

Why kubeadm?
`kubeadm` automates setting up the Kubernetes control plane — generating certificates, configuring etcd, API server, scheduler, and controller manager automatically.

---

### 3.1 Initialize the Cluster

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --ignore-preflight-errors=all

# --pod-network-cidr=10.244.0.0/16 is required for Flannel CNI
# IMPORTANT: Save the "kubeadm join" command shown at the end!
```

---

### 3.2 Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify kubectl works
kubectl cluster-info
kubectl get nodes
# Node will show NotReady until CNI is installed
```

---

### 3.3 Generate Join Command

```bash
# Generate a new join command (valid for 24 hours)
kubeadm token create --print-join-command
```

---

### 3.4 Join Worker Nodes

Run on each worker node using the output from 3.3
```bash
sudo kubeadm join <master-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

### 3.5 Verify Nodes

```bash
# Run on master node
kubectl get nodes
# All nodes show NotReady until CNI is installed in next step
```

---

## 4. CNI Plugin — Flannel

### Why CNI is needed

Without a CNI (Container Network Interface) plugin,  pods cannot communicate with each other across nodes.
Flannel creates an overlay network that allows pods on different nodes to reach each other.

---

### 4.1 Install Weave Net

```bash
# Apply Weave Net manifest
kubectl apply -f \
  https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# Watch Weave pods start on all nodes
kubectl get pods -n kube-system -l name=weave-net -w
```



---

### 4.2 Verify Nodes are Ready

```bash
# Wait 1-2 minutes after installation
kubectl get nodes
```

Expected:

```
NAME       STATUS   ROLES           AGE
master     Ready    control-plane   5m
worker-1   Ready    <none>          4m
worker-2   Ready    <none>          3m
```

---

### 4.3 Verify System Pods

```bash
kubectl get pods -n kube-system
# All pods should be Running
```

Key pods:

* coredns
* etcd
* kube-apiserver
* kube-controller-manager
* kube-scheduler
* kube-proxy

---

## 5. NFS Server Setup

### Why NFS?

MongoDB requires persistent storage that survives pod restarts.
NFS allows all Kubernetes nodes to mount the same shared directory,
enabling dynamically provisioned persistent volumes.

---

### 5.1 On the NFS Server Node

```bash
# Install NFS server
sudo apt update
sudo apt install -y nfs-kernel-server

# Create shared directory
sudo mkdir -p /srv/nfs/medimesh
sudo chown nobody:nogroup /srv/nfs/medimesh
sudo chmod 777 /srv/nfs/medimesh

# Configure NFS export
# rw=read/write, sync=write to disk before confirming
# no_subtree_check=better reliability
# no_root_squash=allow root access from clients
echo "/srv/nfs/medimesh *(rw,sync,no_subtree_check,no_root_squash)" \
  | sudo tee -a /etc/exports

# Apply and enable
sudo exportfs -rav
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server

# Verify
sudo exportfs -v
```

---

### 5.2 On ALL Kubernetes Nodes

Each node needs NFS client tools to mount the NFS share when creating persistent volumes.
```bash
# Install NFS client
sudo apt update
sudo apt install -y nfs-common

# Test the mount works (optional verification)
sudo mount -t nfs <nfs-server-ip>:/srv/nfs/medimesh /mnt
ls /mnt
sudo umount /mnt
echo "NFS mount test successful!"
```

---

## 6. Writing Dockerfiles

### Why these Dockerfile patterns?
Each Dockerfile is optimized for security and size:
* Alpine base image → minimal attack surface
* OS packages upgraded → latest security patches
* Production-only install → no dev dependencies
* npm cache cleaned → smaller image size
* Multi-stage builds → optimized final image

---

### 6.1 Backend Services Dockerfile Pattern

Each backend service follows:

* `FROM node:20-alpine` as base
* `apk update && apk upgrade` for OS security patches
* `npm install --production` for production dependencies only
* Remove npm/yarn/corepack after install
* `EXPOSE` service port
* `CMD ["node", "server.js"]` to start

---

### Service Port Mapping
Port reference for each service:

```
medimesh-auth        → EXPOSE 5001
medimesh-user        → EXPOSE 5002
medimesh-doctor      → EXPOSE 5003
medimesh-appointment → EXPOSE 5004
medimesh-vital       → EXPOSE 5005
medimesh-pharmacy    → EXPOSE 5006
medimesh-ambulance   → EXPOSE 5007
medimesh-complaint   → EXPOSE 5008
medimesh-forum       → EXPOSE 5009
medimesh-bff         → EXPOSE 5010
```

---

### 6.2 Frontend Dockerfile (Multi-Stage)

Already present in `medimesh-frontend/Dockerfile`.
Uses two stages:

Stage 1: `node:20-alpine` — builds the React app
Stage 2: `nginx:alpine` — serves the compiled output
The final image contains NO Node.js or build tools — only Nginx and the compiled HTML/JS/CSS files.

---

### 6.3 Build and Test Locally

Before pushing to CI, test your Docker build locally:
```bash
# Navigate to the service directory
cd medimesh-auth

# Build image locally
docker build -t medimesh-auth:test .

# Run container to test it starts correctly
docker run -d \
  --name auth-test \
  -p 5001:5001 \
  -e MONGO_URI="mongodb://localhost:27017/test" \
  -e JWT_SECRET="testsecret" \
  -e PORT="5001" \
  medimesh-auth:test

# Check the container is running
docker ps

# Test the health endpoint
curl http://localhost:5001/health

# View logs
docker logs auth-test

# Clean up
docker stop auth-test && docker rm auth-test

# Tag for Docker Hub
docker tag medimesh-auth:test \
  bharath44623/medimesh_medimesh-auth:v1.0.0

# Login and push to Docker Hub
docker login -u bharath44623
docker push bharath44623/medimesh_medimesh-auth:v1.0.0
```

---

---

## 7. Kubernetes Namespaces & Helm Charts

### Why Helm?

Helm is the package manager for Kubernetes.
Instead of managing dozens of YAML files, Helm uses templates with values.

---

### Why 3 Namespaces?
MediMesh uses namespace isolation for security.
Each tier has its own namespace with network policies
controlling what can communicate with what.
```text
medimesh-frontend  → React frontend only
medimesh-backend   → All microservices
medimesh-db        → MongoDB + NFS provisioner
```

---

### 7.1 Install Helm

```bash id="helm1"
# Install Helm using official script
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

---

### 7.2 Repository Structure

The `manifest_repo` is the GitOps target that ArgoCD watches.
It contains individual Helm charts for each service:

```text
manifest_repo/
└── helm/
    ├── infrastructure/   ← namespaces, configmap, network-policies, secrets
    ├── auth/             ← Auth service chart (values.yaml has image.tag)
    ├── user/             ← User service chart
    ├── doctor/           ← Doctor service chart
    ├── appointment/      ← Appointment service chart
    ├── vitals/           ← Vitals service chart
    ├── pharmacy/         ← Pharmacy service chart
    ├── ambulance/        ← Ambulance service chart
    ├── complaint/        ← Complaint service chart
    ├── forum/            ← Forum service chart
    ├── bff/              ← BFF service chart
    ├── frontend/         ← Frontend chart (includes Gateway + HPA)
    ├── mongo/            ← MongoDB StatefulSet
    └── nfs/              ← NFS Dynamic Provisioner
```
Each service chart contains:

* `Chart.yaml` — chart metadata
* `values.yaml` — configurable values (image.tag updated by CI)
* `templates/rollout.yaml` — Argo Rollout (Blue/Green) or Deployment
* `templates/service.yaml` — Active + Preview services

---

### 7.3 Key Design Decisions

#### Init Containers

Every backend service has an init container that waits
for MongoDB to be ready before the app starts.
This prevents CrashLoopBackOff on startup when
MongoDB hasn't finished initializing yet

```bash id="init1"
# What the init container does:
# Retries every 3 seconds until MongoDB responds
# Only then does the main application container start
nc -z <mongodb-host> 27017
```

---

#### Health Probes

* `Readiness probe`: Pod only receives traffic when healthy
* `Liveness probe`: Pod is restarted if it becomes unhealthy

---

#### Resource Requests and Limits

All pods have defined resource requests and limits to prevent any one service from consuming all node resources.

---

#### Sealed Secrets

Secrets are stored encrypted in Git using Bitnami Sealed Secrets.
The `medimesh-secrets` secret contains JWT_SECRET and admin credentials.
The `medimesh-mongo-secret` contains MongoDB root credentials.

---

### 7.4 Network Policies (Zero Trust)

The network policy design follows "deny all, allow specific":
```text
Default: ALL traffic DENIED in all namespaces
Then explicitly allow:
  ✅ External → Frontend (gateway ingress)
  ✅ Frontend → BFF (port 5010)
  ✅ BFF → All Backend services (inter-service)
  ✅ Backend → MongoDB (port 27017)
  ✅ MongoDB → MongoDB (internal replication)
  ✅ All → DNS (port 53, UDP+TCP)
  ❌ Frontend → MongoDB (denied)
  ❌ External → Backend directly (denied)
  ❌ External → MongoDB (denied)
```

---

### 7.5 Validate Helm Charts

```bash id="helm2"
cd manifest_repo

# Lint charts to check for errors
helm lint helm/auth
helm lint helm/infrastructure
helm lint helm/mongo

# Dry-run to see what will be created
helm template medimesh-auth helm/auth \
  --debug

# Check rendered templates look correct
helm template medimesh-infrastructure helm/infrastructure \
  | grep "kind:"
```

---

## 8. Sealed Secrets Setup

### Why Sealed Secrets?

Regular Kubernetes Secrets are only base64 encoded (NOT encrypted).
If you commit them to Git, anyone with access can decode them.

Bitnami Sealed Secrets encrypts secrets with a public key.
Only the Sealed Secrets controller in YOUR cluster can decrypt them.
This makes it completely safe to store encrypted secrets in Git.

---

### Flow

```text
Secret → kubeseal → SealedSecret → ArgoCD → Cluster Secret

Plain Secret
    ↓ kubeseal encrypts with cluster public key
SealedSecret YAML (encrypted, safe to commit)
    ↓ ArgoCD deploys to cluster
Sealed Secrets controller decrypts
    ↓
Regular Kubernetes Secret (only exists in cluster)
```

---

### 8.1 Install Controller

```bash id="seal1"
# Install the controller in the cluster
# It holds the private decryption key
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Wait for controller to be ready
kubectl rollout status deployment sealed-secrets-controller \
  -n kube-system

# Verify
kubectl get pods -n kube-system | grep sealed-secrets
```

---

### 8.2 Install kubeseal CLI

```bash id="seal2"
# Download
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz

# Extract
tar -xvzf kubeseal-0.24.0-linux-amd64.tar.gz kubeseal

# Install
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# Verify
kubeseal --version
```

---

### 8.3 Create Application Secret

```bash id="seal3"# Step 1: Create secret in memory only (--dry-run=client)
# This does NOT apply it to the cluster
kubectl create secret generic medimesh-secrets \
  --from-literal=JWT_SECRET="your-super-secret-jwt-key-here" \
  --from-literal=ADMIN_USERNAME="admin" \
  --from-literal=ADMIN_PASSWORD="your-admin-password" \
  --from-literal=MONGO_INITDB_ROOT_USERNAME="mongoroot" \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD="your-mongo-password" \
  --namespace medimesh-backend \
  --dry-run=client \
  -o yaml > medimesh-secrets.yaml

# Step 2: Seal it using cluster's public key
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  < medimesh-secrets.yaml \
  > medimesh-secrets-sealed.yaml

# Step 3: View the sealed file (safe to commit to Git!)
cat medimesh-secrets-sealed.yaml
# Data will look like: AgAZwEPRtu5fil0n+GKUWVezoSzD...

# Step 4: Apply the sealed secret to the cluster
kubectl apply -f medimesh-secrets-sealed.yaml

# Step 5: Verify the controller decrypted it
kubectl get secret medimesh-secrets -n medimesh-backend

# Step 6: Clean up the plain secret file (NEVER commit this!)
rm medimesh-secrets.yaml

# Step 7: Copy sealed file to Helm chart (safe to commit)
cp medimesh-secrets-sealed.yaml \
  manifest_repo/helm/infrastructure/templates/sealed-secrets/medimesh-secrets-sealed.yaml
```

---

### 8.4 MongoDB Sealed Secret

```bash id="seal4"
kubectl create secret generic medimesh-mongo-secret \
  --from-literal=MONGO_INITDB_ROOT_USERNAME="mongoroot" \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD="password" \
  --namespace medimesh-db \
  --dry-run=client -o yaml > mongo-secret.yaml

kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  < mongo-secret.yaml \
  > medimesh-mongo-secret-sealed.yaml

kubectl apply -f medimesh-mongo-secret-sealed.yaml

# Verify
kubectl get secret medimesh-mongo-secret -n medimesh-db

# Clean up and copy to Helm chart
rm mongo-secret.yaml
cp medimesh-mongo-secret-sealed.yaml \
  manifest_repo/helm/infrastructure/templates/sealed-secrets/medimesh-mongo-secret-sealed.yaml

# Commit both sealed secret files to Git (safe!)
cd manifest_repo
git add helm/infrastructure/templates/sealed-secrets/
git commit -m "feat: add sealed secrets"
git push
```

---

## 9. Argo Rollouts Setup

### Why Argo Rollouts?

Standard Kubernetes Deployments use Rolling Update strategy.
Argo Rollouts adds Blue/Green and Canary deployment strategies.

Blue/Green deployment works like this:

Blue = current version actively serving all traffic
Green = new version deployed in parallel, NOT serving traffic
Team tests Green (preview endpoint available)
Human promotes: Green becomes Blue, old Blue terminated
Result: Zero downtime, instant rollback capability

---

---

### 9.1 Install Argo Rollouts

```bash id="roll1"
# Create namespace
kubectl create namespace argo-rollouts

# Install controller
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Wait for controller
kubectl rollout status deployment argo-rollouts \
  -n argo-rollouts

# Verify
kubectl get pods -n argo-rollouts
```

---

### 9.2 Install kubectl Plugin

```bash id="roll2"
# Download plugin
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64

# Install
sudo install -o root -g root -m 0755 \
  kubectl-argo-rollouts-linux-amd64 \
  /usr/local/bin/kubectl-argo-rollouts

# Verify
kubectl argo rollouts version
```

---

### 9.3 Dashboard

```bash id="roll3"
# Start dashboard (optional but very useful visually)
kubectl argo rollouts dashboard &
# Access at: http://localhost:3100
# Shows all rollouts, status, and allows promotion from UI
```

Access: http://localhost:3100

---

### 9.4 Manage Rollouts

```bash id="roll4"
# List all rollouts
kubectl argo rollouts list rollouts -n medimesh-backend

# Get detailed status of a rollout
kubectl argo rollouts get rollout medimesh-auth \
  -n medimesh-backend

# Watch rollout in real time
kubectl argo rollouts get rollout medimesh-auth \
  -n medimesh-backend --watch

# PROMOTE: Switch traffic from Blue to Green
# Run after verifying Green version works correctly
kubectl argo rollouts promote medimesh-auth \
  -n medimesh-backend

# ABORT: Keep Blue, discard Green (instant rollback)
kubectl argo rollouts abort medimesh-auth \
  -n medimesh-backend

# RETRY: Retry a failed rollout
kubectl argo rollouts retry rollout medimesh-auth \
  -n medimesh-backend
```

---

---

## 10. ArgoCD Setup

### Why ArgoCD?

ArgoCD implements GitOps.

Git = single source of truth
Cluster state always matches Git

The GitOps loop:
1. CI pipeline updates image tag in manifest_repo
2. ArgoCD polls manifest_repo every ~3 minutes
3. ArgoCD detects drift (values.yaml changed)
4. ArgoCD applies the new Helm chart to cluster
5. New version deployed — cluster matches Git again
The key benefit: No manual kubectl apply needed.
Update Git → cluster updates automatically and reliably.

---

### GitOps Flow

```text id="gitops1"
CI updates manifest_repo
        ↓
ArgoCD detects change
        ↓
Applies Helm chart
        ↓
Cluster updated automatically
```

---

### 10.1 Install ArgoCD

```bash id="argo1"
# Create namespace
kubectl create namespace argocd

# Install (official manifest)
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods
kubectl wait --for=condition=Ready pods \
  --all -n argocd \
  --timeout=300s

# Verify
kubectl get pods -n argocd
```

---

### 10.2 Access UI

```bash id="argo2"
# Method 1: Port-forward (local access)
kubectl port-forward svc/argocd-server \
  -n argocd 8080:443 &
# Access: https://localhost:8080

# Method 2: NodePort (remote/VM access)
kubectl patch svc argocd-server \
  -n argocd \
  -p '{"spec": {"type": "NodePort"}}'

kubectl get svc argocd-server -n argocd
# Look for port like: 443:32XXX/TCP
# Access: https://<master-ip>:32XXX
```

Access: https://localhost:8080

---

### 10.3 Get Admin Password

```bash id="argo3"
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Login:
# Username: admin
# Password: <output from above>
```

---

### 10.4 Install CLI

```bash id="argo4"
# Download
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

# Install
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Login via CLI
argocd login localhost:8080 \
  --username admin \
  --password <initial-password> \
  --insecure

# Change password (recommended)
argocd account update-password \
  --current-password <initial-password> \
  --new-password <your-new-password>
```

---

### 10.5 Add Repo

ArgoCD needs access to the private `manifest_repo`.
Use a Personal Access Token (PAT) for authentication.
```bash id="argo6"
argocd repo add https://github.com/Medimesh-grp3/manifest_repo \
  --username <username> \
  --password <pat>

# Verify
argocd repo list
```

---

### 10.6 Create ArgoCD Applications

Each Helm chart in manifest_repo becomes one ArgoCD Application.
* `--sync-policy automated` — ArgoCD syncs when Git changes
* `--auto-prune` — removes resources deleted from Git
* `--self-heal` — reverts manual cluster changes back to Git state

```bash id="argo7"
# Infrastructure (deploy FIRST — creates namespaces and secrets)
argocd app create medimesh-infrastructure \
  --repo https://github.com/Medimesh-grp3/manifest_repo \
  --path helm/infrastructure \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace medimesh-backend \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# MongoDB
argocd app create medimesh-mongo \
  --repo https://github.com/Medimesh-grp3/manifest_repo \
  --path helm/mongo \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace medimesh-db \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# NFS Provisioner
argocd app create medimesh-nfs \
  --repo https://github.com/Medimesh-grp3/manifest_repo \
  --path helm/nfs \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace medimesh-db \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# All backend microservices
for service in auth user doctor appointment vitals \
               pharmacy ambulance complaint forum bff; do
  echo "Creating ArgoCD app: medimesh-${service}"
  argocd app create medimesh-${service} \
    --repo https://github.com/Medimesh-grp3/manifest_repo \
    --path helm/${service} \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace medimesh-backend \
    --sync-policy automated \
    --auto-prune \
    --self-heal
done

# Frontend (different namespace!)
argocd app create medimesh-frontend \
  --repo https://github.com/Medimesh-grp3/manifest_repo \
  --path helm/frontend \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace medimesh-frontend \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

---

### 10.7 Verify ArgoCD Applications

```bash id="argo9"
# List all apps and their sync status
argocd app list

# Check specific app
argocd app get medimesh-auth

# Manually trigger sync if needed
argocd app sync medimesh-auth

# View sync history
argocd app history medimesh-auth

# Rollback to previous revision
argocd app rollback medimesh-auth <revision-number>

# Watch all apps
kubectl get applications -n argocd -w
```

---

## 11. kGateway Setup

### Why kGateway?

kGateway implements the Kubernetes Gateway API (next-gen Ingress).
It provides a single external entry point for all MediMesh services.
HTTP routing rules direct requests to the correct microservice
based on URL path prefix:

---

```text id="gw1"
http://<gateway-ip>/auth/*        → medimesh-auth-svc:5001
http://<gateway-ip>/user/*        → medimesh-user-svc:5002
http://<gateway-ip>/doctor/*      → medimesh-doctor-svc:5003
http://<gateway-ip>/appointment/* → medimesh-appointment-svc:5004
http://<gateway-ip>/vitals/*      → medimesh-vitals-svc:5005
http://<gateway-ip>/pharmacy/*    → medimesh-pharmacy-svc:5006
http://<gateway-ip>/ambulance/*   → medimesh-ambulance-svc:5007
http://<gateway-ip>/complaint/*   → medimesh-complaint-svc:5008
http://<gateway-ip>/forum/*       → medimesh-forum-svc:5009
http://<gateway-ip>/api/*         → medimesh-bff-svc:5010
http://<gateway-ip>/             → medimesh-frontend-svc:80
```
The Gateway and HTTPRoute resources are already defined in the manifest_repo/helm/frontend/templates/ directory and are deployed automatically when the frontend Helm chart is applied by ArgoCD.

---

### 11.1 Install Gateway CRDs

```bash id="gw2"
# Install Gateway API Custom Resource Definitions
kubectl apply -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

# Verify CRDs installed
kubectl get crd | grep gateway
```

---

### 11.2 Install kGateway

```bash id="gw3"
# Add Helm repo
helm repo add kgateway-dev \
  https://kgateway-dev.github.io/kgateway/
helm repo update

# Install kGateway
helm install kgateway kgateway-dev/kgateway \
  --namespace kgateway-system \
  --create-namespace \
  --version 2.0.0

# Verify pods
kubectl get pods -n kgateway-system

# Verify GatewayClass registered
kubectl get gatewayclass
# Should show: kgateway   True
```

---

### 11.3 Verify Gateway

The Gateway is deployed automatically as part of the frontend Helm chart. After deploying frontend, verify:
```bash id="gw4"
# Check Gateway has an external IP
kubectl get gateway -n medimesh-frontend

# Check HTTPRoute is accepted
kubectl get httproute -n medimesh-frontend

# Get the external IP
GATEWAY_IP=$(kubectl get gateway medimesh-gateway \
  -n medimesh-frontend \
  -o jsonpath='{.status.addresses[0].value}')

echo "Gateway IP: ${GATEWAY_IP}"

# Test routing works
curl http://${GATEWAY_IP}/auth/health
curl http://${GATEWAY_IP}/
```

---

## 12. Helm Deployment

### Deployment Order
The order matters because components depend on each other:
```text id="order1"
1. NFS Provisioner  → creates StorageClass (MongoDB needs this)
2. MongoDB          → database (all backend services need this)
3. Infrastructure   → namespaces, configmap, network policies, secrets
4. Backend Services → depend on MongoDB + secrets from infrastructure
5. Frontend         → depends on backend services being available
```

---

### 12.1 NFS

```bash id="helm3"
helm install medimesh-nfs helm/nfs \
  -n medimesh-db \
  --create-namespace

kubectl get sc
```

---

### 12.2 MongoDB

```bash id="helm4"
cd manifest_repo

helm install medimesh-nfs helm/nfs \
  -n medimesh-db \
  --create-namespace

# Verify StorageClass created
kubectl get sc
# Should see: nfs-dynamic-medimesh

# Verify NFS provisioner pod running
kubectl get pods -n medimesh-db
```

---

### 12.3 Infrastructure

```bash id="helm5"
helm install medimesh-infrastructure helm/infrastructure \
  -n medimesh-backend \
  --create-namespace

# Verify namespaces created
kubectl get ns | grep medimesh

# Verify network policies applied
kubectl get networkpolicy -n medimesh-frontend
kubectl get networkpolicy -n medimesh-backend
kubectl get networkpolicy -n medimesh-db

# Verify configmap created
kubectl get configmap medimesh-config -n medimesh-backend

# Verify Sealed Secrets controller decrypted the secrets
kubectl get secret medimesh-secrets -n medimesh-backend
kubectl get secret medimesh-mongo-secret -n medimesh-db
```

---

### 12.4 Backend Services

```bash id="helm6"
for service in auth user doctor appointment vitals \
               pharmacy ambulance complaint forum bff; do
  echo "Deploying: medimesh-${service}"
  helm install medimesh-${service} helm/${service} \
    -n medimesh-backend
done

# Watch all pods come up
kubectl get pods -n medimesh-backend -w

# Check for any failing pods
kubectl get pods -n medimesh-backend | grep -v Running
```

---

### 12.5 Frontend

```bash id="helm7"
helm install medimesh-frontend helm/frontend \
  -n medimesh-frontend

# Verify frontend pod running
kubectl get pods -n medimesh-frontend

# Verify services
kubectl get svc -n medimesh-frontend

# Check HPA is created
kubectl get hpa -n medimesh-frontend

# Check Gateway and HTTPRoute
kubectl get gateway -n medimesh-frontend
kubectl get httproute -n medimesh-frontend
```

---

### 12.6 Verification

```bash id="helm8"
echo "=== Frontend Pods ==="
kubectl get pods -n medimesh-frontend

echo "=== Backend Pods ==="
kubectl get pods -n medimesh-backend

echo "=== Database Pods ==="
kubectl get pods -n medimesh-db

echo "=== Rollouts ==="
kubectl argo rollouts list rollouts -n medimesh-backend

echo "=== PVCs ==="
kubectl get pvc -n medimesh-db

echo "=== HPA ==="
kubectl get hpa -n medimesh-frontend
```

---

### 12.7 Helm Upgrade

```bash id="helm9"
# Upgrade a service
helm upgrade medimesh-auth helm/auth \
  -n medimesh-backend

# Check upgrade history
helm history medimesh-auth -n medimesh-backend

# Rollback to previous revision
helm rollback medimesh-auth 1 -n medimesh-backend

# List all releases
helm list -A
```

---


---

## 13. Prometheus & Grafana Setup

### Why Prometheus + Grafana?

* Prometheus → collects metrics
* Grafana → visualizes metrics

Together they provide:
* Real-time cluster health visibility
* Pod CPU and memory usage per service
* Node resource utilization across all nodes
* Kubernetes object state (pods, deployments, rollouts)

---

### 13.1 Add Helm Repo

```bash id="mon1"
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
```

---

### 13.2 Install Monitoring Stack

This single chart installs everything needed for monitoring:
* `Prometheus` — scrapes and stores metrics
* `Grafana` — dashboard and visualization
* `Node Exporter` — system metrics from each node
* `kube-state-metrics` — Kubernetes object metrics
```bash id="mon2"
# Create monitoring namespace
kubectl create namespace monitoring

# Install the full monitoring stack
helm install prometheus \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=7d \
  --set grafana.adminPassword=admin123 \
  --set grafana.service.type=NodePort \
  --set prometheus.service.type=NodePort

# Wait for all pods to be ready
kubectl rollout status deployment prometheus-grafana \
  -n monitoring

# Verify
kubectl get pods -n monitoring
```

---

### 13.3 Access Grafana

```bash id="mon3"
# Method 1: Port-forward
kubectl port-forward svc/prometheus-grafana \
  -n monitoring 3000:80 &
# Access: http://localhost:3000
# Username: admin
# Password: admin123

# Method 2: Get NodePort
kubectl get svc prometheus-grafana -n monitoring
# Access: http://<node-ip>:<nodeport>
```

Access: http://localhost:3000
Login: admin / admin123

---

### 13.4 Access Prometheus

```bash id="mon4"
kubectl port-forward \
  svc/prometheus-kube-prometheus-prometheus \
  -n monitoring 9090:9090 &
# Access: http://localhost:9090
# Go to: Status → Targets to see what is being scraped
```

Access: http://localhost:9090

---

### 13.5 ServiceMonitor

ServiceMonitor tells Prometheus which pods to scrape for metrics.
This makes MediMesh backend pods visible in Prometheus.
```bash id="mon5"
cat > servicemonitor.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: medimesh-backend-monitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames:
      - medimesh-backend
  selector:
    matchLabels:
      tier: backend
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
EOF

kubectl apply -f servicemonitor.yaml

# Verify in Prometheus UI:
# Status → Targets → medimesh-backend targets should appear
```

---

### 13.6 Grafana Dashboards

```text id="mon6"
# In Grafana UI: Left menu → Dashboards → Import

# Recommended Dashboard IDs (enter ID and click Load):
# 1860  → Node Exporter Full
#         (CPU, Memory, Disk, Network per node)
# 6417  → Kubernetes Cluster Overview
# 13770 → Kubernetes All-in-one Cluster Monitoring
# 15661 → Kubernetes Rollouts (Argo Rollouts monitoring)

# Steps:
# 1. Dashboards → Import
# 2. Enter dashboard ID
# 3. Click Load
# 4. Select Prometheus as data source
# 5. Click Import
```

---

### 13.7 Prometheus Queries

```bash id="mon7"
# In Prometheus UI → Graph → Enter these queries:

# Pod CPU usage
rate(container_cpu_usage_seconds_total{namespace="medimesh-backend"}[5m])

# Pod memory usage
container_memory_usage_bytes{namespace="medimesh-backend"}

# Pod restart count (useful for spotting instability)
kube_pod_container_status_restarts_total{namespace="medimesh-backend"}

# Number of running pods per service
kube_pod_status_phase{namespace="medimesh-backend", phase="Running"}

# Node CPU utilization
1 - avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Node memory usage
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

---

## 14. SonarQube Setup

### Why SonarQube?

SonarQube performs Static Application Security Testing (SAST).
It analyzes source code without running it, detecting:

* Bugs — code patterns that cause runtime errors
* Code Smells — maintainability issues and bad practices
* Security Hotspots — potentially vulnerable code patterns
* Duplicated Code — copy-paste issues

In the MediMesh CI pipeline, SonarQube is Stage 1.
If it fails, the entire pipeline stops immediately — no Docker build, no image push, no deployment.
This enforces code quality as a hard gate.

---

### 14.1 PostgreSQL Setup (SonarQube Database)

```bash id="sonar1"
sudo apt update
sudo apt install -y postgresql postgresql-contrib

sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create SonarQube database and user
sudo -u postgres psql << EOF
CREATE USER sonar WITH ENCRYPTED PASSWORD 'sonarpassword';
CREATE DATABASE sonarqube OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;
\q
EOF

# Verify
sudo -u postgres psql -l | grep sonarqube
```


---

### 14.2 Install Java

```bash id="sonar3"
# SonarQube requires Java 17+
sudo apt install -y openjdk-17-jdk
java -version
# Should show: openjdk version "17.x.x"
```

---

### 14.3 Install SonarQube

```bash id="sonar4"
# Download SonarQube Community Edition
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.3.0.82913.zip

sudo apt install -y unzip
unzip sonarqube-10.3.0.82913.zip
sudo mv sonarqube-10.3.0.82913 /opt/sonarqube

# Create dedicated user (SonarQube cannot run as root)
sudo useradd -r -s /bin/false sonar
sudo chown -R sonar:sonar /opt/sonarqube
```

---

### 14.4 Configure SonarQube

```bash id="sonar5"
sudo tee /opt/sonarqube/conf/sonar.properties << EOF
sonar.jdbc.username=sonar
sonar.jdbc.password=sonarpassword
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
sonar.web.host=0.0.0.0
sonar.web.port=9000
sonar.search.javaOpts=-Xmx512m -Xms512m
sonar.web.javaOpts=-Xmx512m -Xms512m
EOF

# System requirements for Elasticsearch (used internally)
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072

# Make permanent (survive reboot)
echo "vm.max_map_count=524288" | sudo tee -a /etc/sysctl.conf
echo "fs.file-max=131072" | sudo tee -a /etc/sysctl.conf

# File descriptor limits for sonar user
sudo tee /etc/security/limits.d/sonar.conf << EOF
sonar   soft    nofile  131072
sonar   hard    nofile  131072
sonar   soft    nproc   8192
sonar   hard    nproc   8192
EOF
```

---

### 14.5 Systemd Service

```bash id="sonar6"
sudo tee /etc/systemd/system/sonarqube.service << EOF
[Unit]
Description=SonarQube service
After=syslog.target network.target postgresql.service

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonar
Group=sonar
Restart=always
LimitNOFILE=131072
LimitNPROC=8192
TimeoutStartSec=5m

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start sonarqube
sudo systemctl enable sonarqube

# Check status
sudo systemctl status sonarqube

# Watch logs — wait for "SonarQube is up" (takes 2-3 minutes)
sudo tail -f /opt/sonarqube/logs/sonar.log
```

---

### 14.6 Initial Setup

Access: http://<ip>:9000

* Login: admin / admin
* Create projects
* Generate token

Save:

```text id="sonar7"
SONARQUBE_TOKEN
SONARQUBE_URL
# Access: http://<sonarqube-ip>:9000
# Default login: admin / admin
# Change password when prompted on first login

# Create one project per microservice:
# Administration → Projects → Create Project → Manually
# Use these EXACT project keys (must match CI workflow):

# medimesh-auth         medimesh-user
# medimesh-doctor       medimesh-appointment
# medimesh-vitals       medimesh-pharmacy
# medimesh-ambulance    medimesh-complaint
# medimesh-forum        medimesh-bff
# medimesh-frontend

# Generate authentication token:
# User Menu (top right) → My Account → Security
# Token Name: github-actions
# Click Generate → COPY the token (shown only once!)
# Save as: SONARQUBE_TOKEN in GitHub Secrets

# Note your SonarQube server URL
# Save as: SONARQUBE_URL = http://<ip>:9000
```

---

## 15. GitHub Actions CI Setup

### CI Flow

Overview of CI Architecture
MediMesh uses a centralized template repository (template-ci)
containing reusable GitHub Actions workflow templates.
Each microservice's CI file is a thin caller that passes
service-specific parameters to these shared templates.

This is the DRY (Don't Repeat Yourself) principle
applied to CI/CD pipelines.

```text id="ci1"
Push → SonarQube → Build → Snyk → Docker → Trivy → Push → Approval → CD

Overview of CI Architecture
MediMesh uses a centralized template repository (template-ci)
containing reusable GitHub Actions workflow templates.
Each microservice's CI file is a thin caller that passes
service-specific parameters to these shared templates.

This is the DRY (Don't Repeat Yourself) principle
applied to CI/CD pipelines.
```

---

### Email Notifications

MediMesh uses Gmail SMTP (via `dawidd6/action-send-mail@v3`)
directly inside GitHub Actions workflows to send emails.
This is NOT Prometheus Alertmanager.

Emails are sent at these pipeline events:
* ❌ Any stage fails → failure alert email
* ⏳ Approval needed → approval request email with link
* ✅ Full pipeline succeeds → success summary email

All email configuration is done via GitHub Secrets:
`SMTP_USERNAME`, `SMTP_PASSWORD`, `ALERT_EMAIL`

```text
Developer pushes code to main branch
          │
          ▼
Stage 1: SonarQube        ← Static code quality analysis
  Pass → continue  |  Fail → email alert + pipeline stops
          │
          ▼
Stage 2: NPM Build        ← npm install + npm run build
  Pass → continue  |  Fail → email alert + pipeline stops
          │
          ▼
Stage 3: Snyk Scan        ← Dependency vulnerability scan
  continue-on-error (advisory only, report uploaded as artifact)
          │
          ▼
Stage 4: Docker Build     ← Build image with 3 semantic version tags
Stage 5: Trivy Scan       ← Container scan (advisory, continue-on-error)
Stage 6: Docker Push      ← Push to Docker Hub (main branch only)
  Pass → continue  |  Fail → email alert + pipeline stops
          │
          ▼
Stage 7: Approval Gate    ← PAUSE pipeline
  Email sent to reviewer with approve/reject link
  Reviewer clicks link → GitHub → Approve and deploy
          │ Approved
          ▼
Stage 8: CD Update        ← Update image tag in manifest_repo
  yq updates helm/<service>/values.yaml
  Git commit + push (with retry)
          │
          ▼
ArgoCD detects change → syncs to cluster → Argo Rollout → Blue/Green
          │
          ▼
Success email sent ✅
```
---

### 15.1 Organization Secrets

Navigate to:
GitHub → Medimesh-grp3 → Settings → Secrets and variables → Actions → New organization secret

```text id="ci2"
SONARQUBE_TOKEN
  → SonarQube UI: My Account → Security → Generate Token → Copy

SONARQUBE_URL
  → http://<sonarqube-server-ip>:9000

SNYK_TOKEN
  → https://app.snyk.io → Account Settings → API Token → Copy

DOCKERHUB_USERNAME
  → bharath44623

DOCKERHUB_TOKEN
  → Docker Hub → Account Settings → Security → New Access Token → Copy

MANIFEST_REPO_PAT
  → GitHub → Settings → Developer Settings →
    Personal access tokens → Fine-grained tokens → New token
    Repository: manifest_repo
    Permissions: Contents → Read and Write
    → Generate → Copy

SMTP_USERNAME
  → your-gmail@gmail.com
  (This is the Gmail address that SENDS the CI/CD notification emails)

SMTP_PASSWORD
  → Gmail App Password (NOT your regular Gmail password)
    Google Account → Security → 2-Step Verification →
    App passwords → Select app: Mail → Generate → Copy

ALERT_EMAIL
  → email address that RECEIVES all CI/CD notifications
```

---

### 15.2 Environment Setup

Create environment: `Prod`
The `Prod` environment causes the pipeline to pause at the Approval Gate stage. Only listed reviewers can approve.

* Required reviewers
* Approval required before deploy

For EACH service repository separately:
```text
Repository → Settings → Environments →
New environment → Name: "Prod" → Configure environment →
Required reviewers → Add your GitHub username →
Save protection rules
```
How approval works:

1. Pipeline reaches approval-gate stage and pauses
2. GitHub Actions sends approval request email to ALERT_EMAIL
3. Email contains a direct link to the pipeline run
4. Reviewer opens the link
5. Clicks "Review deployments" → selects "Prod" → "Approve and deploy"
6. Pipeline continues to CD stage
7. Clicking "Reject" stops the pipeline — nothing is deployed

---

### 15.3 Template Repo

The template workflow files are already written and present
in the `template-ci` repository. Just ensure the repo exists
and all template files are pushed to the main branch:

```bash id="ci3"
cd template-ci

# Verify all template files are present
ls .github/workflows/
# Expected:
# template-sonarqube.yml
# template-build-snyk.yml
# template-docker.yml
# template-approval-gate.yml
# template-cd.yml
# template-oidc-deploy.yml
# template-ci.yml

# Push to GitHub (if not already done)
git add .
git commit -m "feat: add CI/CD templates"
git push -u origin main
```

---

### 15.4 CI Workflow Files in Each Service Repo

Each service already has its CI file at:
`.github/workflows/ci-<service>.yml`

These files call the templates with service-specific parameters.
Here are the key parameters used per service:

```text id="ci5"
medimesh-auth:
  service_name      → medimesh-auth
  sonar_project_key → medimesh-auth
  docker_image_name → medimesh_medimesh-auth
  values_key        → auth

medimesh-user:
  service_name      → medimesh-user
  sonar_project_key → medimesh-user
  docker_image_name → medimesh_medimesh-user
  values_key        → user

medimesh-doctor:
  service_name      → medimesh-doctor
  sonar_project_key → medimesh-doctor
  docker_image_name → medimesh_medimesh-doctor
  values_key        → doctor

medimesh-appointment:
  service_name      → medimesh-appointment
  sonar_project_key → medimesh-appointment
  docker_image_name → medimesh_medimesh-appointment
  values_key        → appointment

medimesh-pharmacy:
  service_name      → medimesh-pharmacy
  sonar_project_key → medimesh-pharmacy
  docker_image_name → medimesh_medimesh-pharmacy
  values_key        → pharmacy

medimesh-ambulance:
  service_name      → medimesh-ambulance
  sonar_project_key → medimesh-ambulance
  docker_image_name → medimesh_medimesh-ambulance
  values_key        → ambulance

medimesh-complaint:
  service_name      → medimesh-complaint
  sonar_project_key → medimesh-complaint
  docker_image_name → medimesh_medimesh-complaint
  values_key        → complaint

medimesh-bff:
  service_name      → medimesh-bff
  sonar_project_key → medimesh-bff
  docker_image_name → medimesh_medimesh-bff
  values_key        → bff

medimesh-frontend:
  service_name      → medimesh-frontend
  sonar_project_key → medimesh-frontend
  docker_image_name → medimesh_medimesh-frontend
  values_key        → frontend

medimesh-vitals (uses OIDC pattern — see Section 17):
  service_name      → medimesh-vitals
  sonar_project_key → medimesh-vitals
  docker_image_name → medimesh_medimesh-vitals
  values_key        → vitals

medimesh-forum (uses Release Gate pattern):
  service_name      → medimesh-forum
  sonar_project_key → medimesh-forum
  docker_image_name → medimesh_medimesh-forum
  values_key        → forum
```

---

### 15.5 Versioning Strategy

The Docker template uses `mathieudutour/github-tag-action@v6.2` to automatically calculate the next version from commit messages:
```text id="ci6"
Commit starts with "feature:" → minor bump
  v1.0.0 → v1.1.0

Commit starts with "breaking:" → major bump
  v1.0.0 → v2.0.0

Any other message → patch bump
  v1.0.0 → v1.0.1
```
Tag format per service: `<service-name>/v<version>`
Example: `medimesh-auth/v1.0.3`

---

### 15.6 Docker Tags
Every successful build produces exactly 3 Docker image tags:
```text id="ci7"
bharath44623/medimesh_medimesh-auth:v1.0.3
  → Exact semantic version (immutable — use for rollback)

bharath44623/medimesh_medimesh-auth:v1.0.3-abc1234
  → Version + short git SHA (full traceability to commit)

bharath44623/medimesh_medimesh-auth:latest
  → Always points to the most recent successful build
```

---

### 15.7 Security Layers

```text id="ci8"
SonarQube → blocks pipeline
Snyk      → advisory
Trivy     → advisory
```

---

### 15.8 Trigger Pipeline

```bash id="ci9"
# Make a change to trigger the CI in any service
cd medimesh-auth

echo "// pipeline test $(date)" >> server.js

git add .
git commit -m "feat: trigger initial CI pipeline test"
git push origin main

# Monitor at:
# https://github.com/Medimesh-grp3/medimesh-auth/actions
# Watch each stage complete in sequence
```

---


---

## 16. GitHub Actions CD Setup

### CD Flow (GitOps)

```text id="cd1"
CI Pipeline completes all stages + Human approves
          │
          ▼
template-cd.yml automatically:
  1. Checks out manifest_repo using MANIFEST_REPO_PAT
  2. Installs yq (YAML processor)
  3. Shows current image tag:
       yq '.image.tag' helm/auth/values.yaml
       → "v1.0.6"
  4. Updates the tag:
       yq -i '.image.tag = "v1.0.7"' helm/auth/values.yaml
  5. Commits:
       "cd(medimesh-auth): image → v1.0.7"
  6. Pushes with retry (up to 5 times with backoff)
          │
          ▼
ArgoCD polls manifest_repo every ~3 minutes
  → Detects: helm/auth/values.yaml changed
  → Renders Helm template with new image tag
  → Applies diff to Kubernetes cluster
          │
          ▼
Argo Rollouts starts Blue/Green deployment:
  → Pulls new image from Docker Hub
  → Starts Green pods (new version)
  → Blue pods continue serving all traffic
  → Waits for human to promote
          │
          ▼
Human runs: kubectl argo rollouts promote medimesh-auth -n medimesh-backend
  → Traffic switches from Blue to Green (zero downtime)
  → Old Blue pods terminated
          │
          ▼
Success notification email sent ✅
```

---

### 16.1 Verify Manifest Update

```bash id="cd2"
cd manifest_repo
git pull origin main

# Check recent commits
git log --oneline -5
# Should see: cd(medimesh-auth): image → v1.X.X

# Verify the tag was updated
cat helm/auth/values.yaml | grep tag
# Should show: tag: "v1.X.X"
```

---

### 16.2 Verify ArgoCD Sync

```bash id="cd3"
# Check sync status
argocd app get medimesh-auth | grep -i sync
# Status should be: Synced

# If OutOfSync, wait ~3 minutes or manually trigger:
argocd app sync medimesh-auth
```

---

### 16.3 Complete Deployment

```bash id="cd4"
# Check new pod is running
kubectl get pods -n medimesh-backend | grep auth

# Check rollout status
kubectl argo rollouts get rollout medimesh-auth \
  -n medimesh-backend

# Test the preview (Green) version before promoting
kubectl port-forward svc/medimesh-auth-svc-preview \
  -n medimesh-backend 5001:5001 &
curl http://localhost:5001/health
# Expected: {"status":"ok"}

# Promote Green to Blue (zero downtime switch)
kubectl argo rollouts promote medimesh-auth \
  -n medimesh-backend

# Watch the promotion complete
kubectl argo rollouts get rollout medimesh-auth \
  -n medimesh-backend --watch
```

---

### 16.4 Git Retry Logic

The CD template includes retry logic for the git push step. This handles race conditions when multiple services update `manifest_repo` simultaneously (which causes push conflicts):
```bash id="cd5"
# What happens internally in template-cd.yml:
for i in 1 2 3 4 5; do
  git push && break
  echo "Push failed, retrying ($i/5)..."
  git pull --rebase origin main   # Pull latest, rebase our commit on top
  sleep $((i * 2))               # Wait 2s, 4s, 6s, 8s, 10s
done
```

---

## 17. OIDC Setup for Vitals

### Why OIDC?

No static credentials.
Short-lived secure authentication.

The Vitals service demonstrates a more advanced and secure
deployment approach — GitHub OIDC (OpenID Connect)
direct Kubernetes deployment, without ArgoCD or
any static stored credentials.

Problems with traditional kubeconfig approach:
* Store kubeconfig as GitHub Secret → static credential
* If the secret leaks, the cluster is permanently compromised
* Long-lived tokens are a security anti-pattern

---

### OIDC Flow

```text id="oidc1"
GitHub → OIDC Token → K8s API → RBAC → Deploy → Token expires

GitHub Actions workflow starts
  ↓
Requests OIDC JWT token from GitHub's token endpoint
  (Token proves: "I am the workflow from
   Medimesh-grp3/medimesh-vital, branch: main")
  ↓
Presents JWT to Kubernetes API server
  ↓
K8s validates JWT against GitHub's OIDC issuer URL
  (https://token.actions.githubusercontent.com)
  ↓
RBAC checks: Is this identity allowed to patch rollouts?
  Yes → Subject: "github:repo:Medimesh-grp3/medimesh-vital:ref:refs/heads/main"
  ↓
kubectl argo rollouts set image medimesh-vitals ...
  ↓
JWT expires — no lingering credentials!
```

---

### 17.1 Configure API Server

Edit:

```bash id="oidc2"
# Edit kube-apiserver static pod manifest on master
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml

# Find the command section and add these 5 lines:
# - --oidc-issuer-url=https://token.actions.githubusercontent.com
# - --oidc-client-id=kubernetes
# - --oidc-username-claim=sub
# - --oidc-username-prefix=github:
# - --oidc-groups-claim=repository

# The command section should look like:
# spec:
#   containers:
#   - command:
#     - kube-apiserver
#     - --advertise-address=...      (existing flags)
#     - --oidc-issuer-url=https://token.actions.githubusercontent.com
#     - --oidc-client-id=kubernetes
#     - --oidc-username-claim=sub
#     - --oidc-username-prefix=github:
#     - --oidc-groups-claim=repository

# The API server restarts automatically (it's a static pod)
# Wait 60 seconds for it to come back up
sleep 60

# Verify API server is back
kubectl get nodes
```

---

### 17.2 Apply RBAC

The RBAC file is already written and present in the repo at:
`manifest_repo/rbac/github-actions-vitals-rbac.yaml`

This file defines:
* A Role that allows ONLY: get/list/watch/patch/update on `rollouts` and `rollouts/status` in medimesh-backend
* A RoleBinding that grants this role to the OIDC identity: `github:repo:Medimesh-grp3/medimesh-vital:ref:refs/heads/main`

```bash id="oidc4"
# Apply the pre-written RBAC
kubectl apply -f manifest_repo/rbac/github-actions-vitals-rbac.yaml

# Verify role created
kubectl get role github-actions-vitals-deployer \
  -n medimesh-backend

# Verify rolebinding created
kubectl get rolebinding github-actions-vitals-deployer \
  -n medimesh-backend
```

---

### 17.3 Get CA Cert

The CA certificate allows GitHub Actions to verify it is connecting to the correct Kubernetes cluster (prevents MITM).
```bash id="oidc5"
# Extract base64-encoded CA certificate
kubectl config view \
  --raw \
  --minify \
  --output 'jsonpath={.clusters[0].cluster.certificate-authority-data}'

# Copy the full base64 output string
# Save it as GitHub Secret: K8S_CA_CERT
# Keep it base64 encoded — the workflow decodes it with: base64 -d
```

---

### 17.4 Get API Server

```bash id="oidc6"
kubectl config view \
  --minify \
  --output 'jsonpath={.clusters[0].cluster.server}'

# Example output: https://10.0.0.1:6443
# Save as GitHub Secret: K8S_API_SERVER
```

---

### 17.5 Self-Hosted Runner

The OIDC deployment job uses `runs-on: self-hosted` because it needs direct network access to the Kubernetes API server.

```bash id="oidc7"
# In GitHub:
# medimesh-vital repo → Settings → Actions → Runners →
# New self-hosted runner → Linux → x64
# Follow all instructions shown in GitHub UI exactly

# On the master node:
mkdir actions-runner && cd actions-runner

# Download runner (use URL shown in GitHub UI)
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz

tar xzf ./actions-runner-linux-x64.tar.gz

# Configure using the token from GitHub UI (valid 1 hour)
./config.sh \
  --url https://github.com/Medimesh-grp3/medimesh-vital \
  --token <TOKEN_FROM_GITHUB_UI>

# Install as systemd service (runs on boot automatically)
sudo ./svc.sh install
sudo ./svc.sh start

# Verify it's running
sudo ./svc.sh status

# Confirm in GitHub UI:
# Settings → Actions → Runners → should show green dot: Active
```

---

### 17.6 Repo Secrets

```text id="oidc8"
# In medimesh-vital repository:
# Settings → Secrets and variables → Actions →
# New repository secret

# Add these two secrets:
# K8S_CA_CERT   → paste the base64 output from step 17.3
# K8S_API_SERVER → paste the URL from step 17.4
```

---

### 17.7 Promote Vitals

After the OIDC deploy job runs successfully:
* New image is set on the Vitals Argo Rollout
* Green pods start with the new version
* Blue pods continue serving live traffic
* Manual promotion required (same as all other services)

```bash id="oidc9"
# Check vitals rollout status
kubectl argo rollouts get rollout medimesh-vitals \
  -n medimesh-backend

# Test Green (preview) endpoint
kubectl port-forward svc/medimesh-vitals-svc-preview \
  -n medimesh-backend 5005:5005 &
curl http://localhost:5005/health

# Promote if healthy
kubectl argo rollouts promote medimesh-vitals \
  -n medimesh-backend
```

---

## 18. Full Pipeline Verification

### 18.1 End-to-End Test

```bash id="verify1"
# ── Step 1: Make a code change ──────────────────────────
cd medimesh-auth
echo "// pipeline test $(date)" >> server.js

# ── Step 2: Commit and push ──────────────────────────────
git add .
git commit -m "feat: trigger full pipeline test"
git push origin main

# ── Step 3: Monitor GitHub Actions ──────────────────────
# Open browser:
# https://github.com/Medimesh-grp3/medimesh-auth/actions
# Watch each stage complete in sequence

# ── Step 4: Check SonarQube result ──────────────────────
# Open: http://<sonarqube-ip>:9000
# Project: medimesh-auth → new analysis appears

# ── Step 5: Check Snyk report artifact ──────────────────
# In GitHub Actions run → Artifacts section (bottom of page)
# Download: snyk-report-medimesh-auth
# Open snyk-report.html in browser for readable report

# ── Step 6: Check Docker Hub ────────────────────────────
# https://hub.docker.com/r/bharath44623/medimesh_medimesh-auth/tags
# New tags should appear: v1.X.X, v1.X.X-<sha>, latest

# ── Step 7: Respond to approval email ───────────────────
# Check inbox of ALERT_EMAIL
# Email subject: "Approval Needed — medimesh-auth → v1.X.X"
# Click the link in the email
# GitHub → Review deployments → Prod → Approve and deploy

# ── Step 8: Verify manifest_repo updated ────────────────
cd manifest_repo
git pull origin main
git log --oneline -3
# Should see: cd(medimesh-auth): image → v1.X.X

cat helm/auth/values.yaml | grep tag
# Should show: tag: "v1.X.X"

# ── Step 9: Check ArgoCD synced ─────────────────────────
argocd app get medimesh-auth
# Sync Status: Synced
# Health Status: Healthy

# ── Step 10: Check rollout started ──────────────────────
kubectl argo rollouts get rollout medimesh-auth \
  -n medimesh-backend
# Should show: Paused (waiting for promotion)

# ── Step 11: Test Green (preview) version ───────────────
kubectl port-forward svc/medimesh-auth-svc-preview \
  -n medimesh-backend 5001:5001 &
curl http://localhost:5001/health
# Expected: {"status":"ok"}

# ── Step 12: Promote Green → Blue ───────────────────────
kubectl argo rollouts promote medimesh-auth \
  -n medimesh-backend

# Watch promotion complete
kubectl argo rollouts get rollout medimesh-auth \
  -n medimesh-backend --watch

# ── Step 13: Check success email ────────────────────────
# Check inbox — email with subject:
# "MediMesh CI/CD Passed — medimesh-auth (v1.X.X)"
# Contains summary of all 8 stages
```

---

### 18.2 Health Checks

```bash id="verify2"
# Check all pods are Running
kubectl get pods -n medimesh-frontend
kubectl get pods -n medimesh-backend
kubectl get pods -n medimesh-db

# Test via Gateway IP
GATEWAY_IP=$(kubectl get gateway medimesh-gateway \
  -n medimesh-frontend \
  -o jsonpath='{.status.addresses[0].value}')

echo "Gateway IP: ${GATEWAY_IP}"

# Test each service through the gateway
curl http://${GATEWAY_IP}/auth/health
curl http://${GATEWAY_IP}/user/health
curl http://${GATEWAY_IP}/doctor/health
curl http://${GATEWAY_IP}/appointment/health
curl http://${GATEWAY_IP}/vitals/health
curl http://${GATEWAY_IP}/pharmacy/health
curl http://${GATEWAY_IP}/ambulance/health
curl http://${GATEWAY_IP}/complaint/health
curl http://${GATEWAY_IP}/forum/health
curl http://${GATEWAY_IP}/api/health
curl http://${GATEWAY_IP}/
```


---

### 18.3 Monitoring

```bash id="verify4"
# All monitoring pods running
kubectl get pods -n monitoring

# Access Grafana
kubectl port-forward svc/prometheus-grafana \
  -n monitoring 3000:80 &
# http://localhost:3000 → admin / admin123
# Check dashboards are loading with live data

# Access Prometheus
kubectl port-forward \
  svc/prometheus-kube-prometheus-prometheus \
  -n monitoring 9090:9090 &
# http://localhost:9090
# Status → Targets → all should be UP
```

---

### 18.4 Full System Status

```bash id="verify5"echo "════════════════════════════════════════"
echo "  MediMesh Complete System Status"
echo "════════════════════════════════════════"

echo ""
echo "── Kubernetes Nodes ─────────────────"
kubectl get nodes

echo ""
echo "── Frontend Pods ────────────────────"
kubectl get pods -n medimesh-frontend

echo ""
echo "── Backend Pods ─────────────────────"
kubectl get pods -n medimesh-backend

echo ""
echo "── Database Pods ────────────────────"
kubectl get pods -n medimesh-db

echo ""
echo "── Argo Rollouts ────────────────────"
kubectl argo rollouts list rollouts -n medimesh-backend

echo ""
echo "── ArgoCD Applications ──────────────"
argocd app list

echo ""
echo "── Monitoring Pods ──────────────────"
kubectl get pods -n monitoring

echo ""
echo "── Persistent Volume Claims ─────────"
kubectl get pvc -n medimesh-db

echo ""
echo "── Horizontal Pod Autoscaler ────────"
kubectl get hpa -n medimesh-frontend

echo ""
echo "── Network Policies ─────────────────"
kubectl get networkpolicy --all-namespaces

echo ""
echo "── Sealed Secrets ───────────────────"
kubectl get sealedsecrets --all-namespaces

echo ""
echo "── Gateway ──────────────────────────"
kubectl get gateway -n medimesh-frontend
kubectl get httproute -n medimesh-frontend

echo ""
echo "════════════════════════════════════════"
echo "  Status Check Complete ✅"
echo "════════════════════════════════════════"
```

---

## 19. Troubleshooting Reference

### 19.1 Pods Not Starting — CrashLoopBackOff

Symptom: Pods show `CrashLoopBackOff` or `Error` status

```bash id="tr1"# Step 1: Check pod events (most useful first step)
kubectl describe pod <pod-name> -n medimesh-backend
# Look at the "Events" section at the bottom

# Step 2: Check application logs
kubectl logs <pod-name> -n medimesh-backend

# Step 3: Check logs from previous crashed container
kubectl logs <pod-name> -n medimesh-backend --previous

# Step 4: Check init container logs
# (wait-for-mongodb runs before the main app)
kubectl logs <pod-name> \
  -c wait-for-mongodb \
  -n medimesh-backend

# Step 5: Check MongoDB is healthy
kubectl get pods -n medimesh-db
kubectl exec -it medimesh-mongodb-0 \
  -n medimesh-db \
  -- mongosh --eval "rs.status()"

# Common causes and fixes:
# MongoDB not ready    → Wait for all 3 DB pods, check rs.status()
# Wrong MONGO_URI      → Check configmap values match actual hosts
# JWT_SECRET missing   → Check sealed secret was decrypted correctly
# Port conflict        → Look for duplicate pods or port mismatch
```

---

### 19.2 PVC Stuck in Pending

Symptom: PVCs show Pending, MongoDB pods won't start

```bash id="tr2"
# Check PVC status and events
kubectl get pvc -n medimesh-db
kubectl describe pvc <pvc-name> -n medimesh-db

# Check NFS provisioner
kubectl get pods -n medimesh-db | grep nfs
kubectl logs -n medimesh-db -l app=nfs-client-provisioner

# Test NFS mount from a worker node
ssh ubuntu@<worker-node-ip>
sudo mount -t nfs <nfs-ip>:/srv/nfs/medimesh /mnt
ls /mnt
sudo umount /mnt
# If mount fails, the problem is NFS connectivity

# Fix NFS server
ssh ubuntu@<nfs-server-ip>
sudo systemctl status nfs-kernel-server
sudo systemctl restart nfs-kernel-server
sudo exportfs -v

# Install nfs-common if missing from worker nodes
ssh ubuntu@<worker-node-ip>
sudo apt install -y nfs-common
```

---

### 19.3 ArgoCD Not Syncing

Symptom: Apps stay OutOfSync, changes not deploying
  
```bash id="tr3"
# Check app status
argocd app get medimesh-auth

# Force refresh (clears ArgoCD cache)
argocd app get medimesh-auth --refresh

# Force sync
argocd app sync medimesh-auth --force

# Check application controller logs
kubectl logs -n argocd \
  -l app.kubernetes.io/name=argocd-application-controller \
  | tail -30

# Re-add repo if credentials expired
argocd repo list
argocd repo add https://github.com/Medimesh-grp3/manifest_repo \
  --username <username> \
  --password <new-pat>
```

---

### 19.4 GitHub Actions Pipeline Failing

Check:

* GitHub Actions logs
* Secrets configuration
* Tokens

```bash
# Always check the GitHub Actions run first:
# https://github.com/Medimesh-grp3/<service>/actions
# Click failed run → Click failed job → Read error message

# Common issues:

# "SONARQUBE_TOKEN: secret not found"
# → Check organization secrets are configured
# → Verify repo has access to org-level secrets

# "Connection refused to SonarQube"
# → Verify SONARQUBE_URL = http://<ip>:9000 (not https)
# → Port 9000 must be open/reachable from GitHub Actions runners
# → Test: curl http://<sonarqube-ip>:9000/api/system/status

# "Docker push: unauthorized"
# → DOCKERHUB_TOKEN may have expired
# → Create new token in Docker Hub → update GitHub Secret

# "manifest_repo push: 403 Permission denied"
# → MANIFEST_REPO_PAT expired or lacks Contents write permission
# → Create new fine-grained PAT with Contents: Read+Write

# "Waiting for a runner" (Vitals only)
# → Self-hosted runner is not active
# → ssh master → cd actions-runner → sudo ./svc.sh start

# "Environment Prod not found"
# → Create "Prod" environment in repo settings
# → Settings → Environments → New environment → Prod
# → Add required reviewers → Save

# "Action failed for dawidd6/action-send-mail"
# → Check SMTP_USERNAME and SMTP_PASSWORD secrets
# → SMTP_PASSWORD must be Gmail App Password, not account password
# → Verify 2-Step Verification is enabled on Gmail account
```
---

### 19.5 Sealed Secrets Issues

Symptom: SealedSecret exists but regular Secret not created
```bash id="tr4"
# Check if controller is running
kubectl get pods -n kube-system | grep sealed-secrets

# Check controller logs for errors
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=sealed-secrets | tail -20

# Describe the SealedSecret resource
kubectl describe sealedsecret medimesh-secrets \
  -n medimesh-backend

# If error: "no key could decrypt secret"
# Controller was reinstalled — new private key generated
# Must re-seal all secrets with new public key:

# Get new public key from the controller
kubeseal --fetch-cert \
  --controller-namespace kube-system \
  --controller-name sealed-secrets-controller \
  > new-public-cert.pem

# Re-create and re-seal secrets:
kubectl create secret generic medimesh-secrets \
  --from-literal=JWT_SECRET="..." \
  --from-literal=ADMIN_USERNAME="..." \
  --from-literal=ADMIN_PASSWORD="..." \
  --from-literal=MONGO_INITDB_ROOT_USERNAME="..." \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD="..." \
  --namespace medimesh-backend \
  --dry-run=client -o yaml \
  | kubeseal --cert new-public-cert.pem --format yaml \
  > medimesh-secrets-sealed.yaml

kubectl apply -f medimesh-secrets-sealed.yaml

# Verify regular secret was created
kubectl get secret medimesh-secrets -n medimesh-backend
```

---

### 19.6 Blue/Green Rollout Stuck

Symptom: Rollout stays in Paused state indefinitely
```bash id="tr5"# Get detailed rollout status
kubectl argo rollouts get rollout medimesh-auth \
  -n medimesh-backend

# Check rollout events
kubectl describe rollout medimesh-auth \
  -n medimesh-backend

# Check preview pods are healthy
kubectl get pods -n medimesh-backend \
  -l app=medimesh-auth

# Test the preview endpoint
kubectl port-forward svc/medimesh-auth-svc-preview \
  -n medimesh-backend 5001:5001 &
curl http://localhost:5001/health

# If healthy — promote to production
kubectl argo rollouts promote medimesh-auth \
  -n medimesh-backend

# If unhealthy — abort, Blue keeps serving traffic
kubectl argo rollouts abort medimesh-auth \
  -n medimesh-backend

# Force restart if completely stuck
kubectl argo rollouts restart medimesh-auth \
  -n medimesh-backend

# Check Argo Rollouts controller health
kubectl get pods -n argo-rollouts
kubectl logs -n argo-rollouts \
  -l app.kubernetes.io/name=argo-rollouts | tail -20
```

---

### 19.7 Network Policy Blocking Traffic

Symptom: Services cannot reach each other or MongoDB
```bash id="tr6"
# Get a shell in a backend pod to test connectivity
kubectl exec -it \
  $(kubectl get pod -n medimesh-backend \
    -l app=medimesh-bff \
    -o jsonpath='{.items[0].metadata.name}') \
  -n medimesh-backend -- sh

# Inside the pod, test DNS resolution
nslookup medimesh-auth-svc.medimesh-backend.svc.cluster.local
nslookup medimesh-mongodb.medimesh-db.svc.cluster.local

# Test inter-service HTTP
wget -qO- http://medimesh-auth-svc:5001/health

# Test MongoDB port connectivity
nc -z medimesh-mongodb.medimesh-db.svc.cluster.local 27017 \
  && echo "MongoDB: CONNECTED" \
  || echo "MongoDB: BLOCKED"

exit

# Check network policies
kubectl get networkpolicy -n medimesh-backend
kubectl describe networkpolicy allow-bff-to-backend \
  -n medimesh-backend

# CRITICAL: Check namespace labels
# Network policies use namespaceSelector with these labels
kubectl get namespace medimesh-backend --show-labels
kubectl get namespace medimesh-frontend --show-labels
kubectl get namespace medimesh-db --show-labels

# If labels are missing, add them:
kubectl label namespace medimesh-backend \
  medimesh/tier=backend --overwrite
kubectl label namespace medimesh-frontend \
  medimesh/tier=frontend --overwrite
kubectl label namespace medimesh-db \
  medimesh/tier=database --overwrite
```

---

### 19.8 SonarQube Issues in CI

Symptom: SonarQube stage fails or times out in pipeline

```bash id="tr7"
# Check SonarQube server is running
sudo systemctl status sonarqube

# Check logs for errors
sudo tail -100 /opt/sonarqube/logs/web.log
sudo tail -100 /opt/sonarqube/logs/sonar.log

# Test API is accessible
curl http://<sonarqube-ip>:9000/api/system/status
# Expected: {"status":"UP","version":"10.x.x"}

# Test authentication token
curl -u <your-token>: \
  http://<sonarqube-ip>:9000/api/authentication/validate
# Expected: {"valid":true}

# GitHub Actions must be able to reach SonarQube
# (GitHub runners have public IPs — SonarQube must be internet-accessible)
# Open port 9000 in your firewall/security group

# Restart SonarQube if it has crashed or hung
sudo systemctl restart sonarqube
# Wait 2-3 minutes for full startup
sudo tail -f /opt/sonarqube/logs/sonar.log
# Wait for: SonarQube is up

# If project key mismatch error:
# The sonar_project_key in CI must exactly match the
# project key created in SonarQube UI
# Check: SonarQube → Projects → <project> → Project Information
```

## Quick Reference Commands

```bash
# ── Cluster Health ────────────────────────────────────────
kubectl get nodes                                    # Node status
kubectl get pods --all-namespaces                    # All pods
kubectl top nodes                                    # CPU/Memory per node
kubectl top pods -n medimesh-backend                 # CPU/Memory per pod

# ── MediMesh Pods ─────────────────────────────────────────
kubectl get pods -n medimesh-frontend
kubectl get pods -n medimesh-backend
kubectl get pods -n medimesh-db

# ── Logs ──────────────────────────────────────────────────
kubectl logs -f <pod> -n medimesh-backend            # Follow logs
kubectl logs <pod> --previous -n medimesh-backend    # Previous crash
kubectl logs <pod> -c wait-for-mongodb \
  -n medimesh-backend                                # Init container
kubectl logs <pod> -c log-sidecar \
  -n medimesh-frontend                               # Nginx sidecar

# ── Blue/Green Rollouts ───────────────────────────────────
kubectl argo rollouts list rollouts \
  -n medimesh-backend                                # List all
kubectl argo rollouts get rollout <name> \
  -n medimesh-backend                                # Status
kubectl argo rollouts get rollout <name> \
  -n medimesh-backend --watch                        # Watch live
kubectl argo rollouts promote <name> \
  -n medimesh-backend                                # Promote green
kubectl argo rollouts abort <name> \
  -n medimesh-backend                                # Keep blue
kubectl argo rollouts restart <name> \
  -n medimesh-backend                                # Full restart

# ── ArgoCD ────────────────────────────────────────────────
argocd app list                                      # All apps
argocd app get <name>                                # App details
argocd app sync <name>                               # Force sync
argocd app sync <name> --force                       # Hard sync
argocd app history <name>                            # Sync history
argocd app rollback <name> <revision>                # Rollback

# ── Helm ──────────────────────────────────────────────────
helm list -A                                         # All releases
helm status <release> -n <namespace>                 # Release status
helm upgrade <release> helm/<chart> \
  -n <namespace>                                     # Upgrade
helm rollback <release> 1 -n <namespace>             # Rollback
helm history <release> -n <namespace>                # History

# ── MongoDB ───────────────────────────────────────────────
kubectl exec -it medimesh-mongodb-0 \
  -n medimesh-db \
  -- mongosh --eval "rs.status()"                    # RS status
kubectl exec -it medimesh-mongodb-0 \
  -n medimesh-db \
  -- mongosh --eval "db.adminCommand('ping')"        # Quick ping

# ── Sealed Secrets ────────────────────────────────────────
kubectl get sealedsecrets --all-namespaces
kubeseal --fetch-cert \
  --controller-namespace kube-system \
  --controller-name sealed-secrets-controller \
  > cert.pem                                         # Get public cert

# ── Storage ───────────────────────────────────────────────
kubectl get pvc -n medimesh-db                       # PVC status
kubectl get sc                                       # StorageClasses
kubectl get pv                                       # PersistentVolumes

# ── Monitoring ────────────────────────────────────────────
kubectl get pods -n monitoring
kubectl port-forward svc/prometheus-grafana \
  -n monitoring 3000:80 &                            # Grafana
kubectl port-forward \
  svc/prometheus-kube-prometheus-prometheus \
  -n monitoring 9090:9090 &                          # Prometheus

# ── Network Policies ──────────────────────────────────────
kubectl get networkpolicy --all-namespaces
kubectl describe networkpolicy <name> -n <namespace>

# ── Gateway ───────────────────────────────────────────────
kubectl get gateway -n medimesh-frontend
kubectl get httproute -n medimesh-frontend
kubectl get svc -n medimesh-frontend

# ── Debug Shell ───────────────────────────────────────────
kubectl exec -it <pod> -n medimesh-backend -- sh
kubectl exec -it <pod> -n medimesh-backend \
  -- wget -qO- http://medimesh-auth-svc:5001/health
```

---

## ✅ Summary

```text id="summary1"
Infrastructure        → Kubernetes Cluster
Storage              → NFS + PVC
Deployment           → Helm + ArgoCD
Strategy             → Blue/Green (Argo Rollouts)
CI/CD                → GitHub Actions (8 stages)
Security             → SonarQube + Snyk + Trivy
Monitoring           → Prometheus + Grafana
Secrets              → Sealed Secrets
Advanced Deploy      → OIDC (Vitals)
```

---

### Final Stats

```text id="summary2"
Microservices:        11
CI Stages:            8
Security Layers:      3
Namespaces:           3
Deployment Types:     Blue/Green + Rolling
Monitoring:           Enabled
```



