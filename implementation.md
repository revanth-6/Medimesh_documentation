# MediMesh — Complete Implementation Walkthrough

## From Zero to Production: Full DevOps Implementation Guide

**Project:** MediMesh — Hospital Management Microservices Platform
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

MediMesh runs on a self-managed Kubernetes cluster built
using `kubeadm`. The cluster has one master (control plane)
node and two worker nodes. A separate NFS server provides
persistent storage for MongoDB.

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

Kubernetes requires specific kernel modules, network settings,
and a container runtime before the cluster can be formed.
Swap must be disabled because Kubernetes memory management
conflicts with Linux swap behavior.

---

### 2.1 System Update

```bash
sudo apt-get update -y
sudo apt-get upgrade -y
```

---

### 2.2 Disable Swap

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

* overlay — used by containerd for layered filesystems
* br_netfilter — allows iptables to see bridged traffic

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

lsmod | grep overlay
lsmod | grep br_netfilter
```

---

### 2.4 Set Kernel Parameters

```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

sysctl net.ipv4.ip_forward
```

---

### 2.5 Install Container Runtime (containerd)

```bash
sudo apt-get install -y \
  curl \
  gnupg2 \
  software-properties-common \
  apt-transport-https \
  ca-certificates

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor \
  -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt-get update -y
sudo apt-get install -y containerd.io

containerd config default \
  | sudo tee /etc/containerd/config.toml >/dev/null 2>&1

sudo sed -i \
  's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

### 2.6 Install Kubernetes Components

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key \
  | sudo gpg --dearmor \
  -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo \
  'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable kubelet
```

---

## 3. Cluster Initialization

⚠️ Run this section ONLY on the master node

---

### 3.1 Initialize the Cluster

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --ignore-preflight-errors=all
```

---

### 3.2 Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl cluster-info
kubectl get nodes
```

---

### 3.3 Generate Join Command

```bash
kubeadm token create --print-join-command
```

---

### 3.4 Join Worker Nodes

```bash
sudo kubeadm join <master-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

### 3.5 Verify Nodes

```bash
kubectl get nodes
```

---

## 4. CNI Plugin — Flannel

### Why CNI is needed

Without a CNI (Container Network Interface) plugin,
pods cannot communicate with each other across nodes.

Flannel creates an overlay network that allows pods
on different nodes to reach each other.

---

### 4.1 Install Flannel

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

kubectl get pods -n kube-flannel -w
```

---

### 4.2 Verify Nodes are Ready

```bash
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
sudo apt update
sudo apt install -y nfs-kernel-server

sudo mkdir -p /srv/nfs/medimesh
sudo chown nobody:nogroup /srv/nfs/medimesh
sudo chmod 777 /srv/nfs/medimesh

echo "/srv/nfs/medimesh *(rw,sync,no_subtree_check,no_root_squash)" \
  | sudo tee -a /etc/exports

sudo exportfs -rav
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server

sudo exportfs -v
```

---

### 5.2 On ALL Kubernetes Nodes

```bash
sudo apt update
sudo apt install -y nfs-common

sudo mount -t nfs <nfs-server-ip>:/srv/nfs/medimesh /mnt
ls /mnt
sudo umount /mnt

echo "NFS mount test successful!"
```

---

## 6. Writing Dockerfiles

### Why these Dockerfile patterns?

* Alpine base image → minimal attack surface
* OS packages upgraded → latest security patches
* Production-only install → no dev dependencies
* npm cache cleaned → smaller image size
* Multi-stage builds → optimized final image

---

### 6.1 Backend Services Dockerfile Pattern

Each backend service follows:

* `FROM node:20-alpine`
* `apk update && apk upgrade`
* `npm install --production`
* Remove unnecessary tools
* Expose service port
* Start with `node server.js`

---

### Service Port Mapping

```
medimesh-auth        → 5001
medimesh-user        → 5002
medimesh-doctor      → 5003
medimesh-appointment → 5004
medimesh-vital       → 5005
medimesh-pharmacy    → 5006
medimesh-ambulance   → 5007
medimesh-complaint   → 5008
medimesh-forum       → 5009
medimesh-bff         → 5010
```

---

### 6.2 Frontend Dockerfile (Multi-Stage)

Two stages:

Stage 1:

* node:20-alpine
* installs dependencies
* builds React app

Stage 2:

* nginx:alpine
* serves static build

Final image contains:

* NO Node.js
* ONLY Nginx + static files

---

### 6.3 Build and Test Locally

```bash
cd medimesh-auth

docker build -t medimesh-auth:test .

docker run -d \
  --name auth-test \
  -p 5001:5001 \
  -e MONGO_URI="mongodb://localhost:27017/test" \
  -e JWT_SECRET="testsecret" \
  -e PORT="5001" \
  medimesh-auth:test

docker ps

curl http://localhost:5001/health

docker logs auth-test

docker stop auth-test && docker rm auth-test

docker tag medimesh-auth:test \
  bharath44623/medimesh_medimesh-auth:v1.0.0

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

```text
medimesh-frontend  → React UI
medimesh-backend   → All microservices
medimesh-db        → MongoDB + storage
```

---

### 7.1 Install Helm

```bash id="helm1"
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version
```

---

### 7.2 Repository Structure

```text
manifest_repo/
└── helm/
    ├── infrastructure/
    ├── auth/
    ├── user/
    ├── doctor/
    ├── appointment/
    ├── vitals/
    ├── pharmacy/
    ├── ambulance/
    ├── complaint/
    ├── forum/
    ├── bff/
    ├── frontend/
    ├── mongo/
    └── nfs/
```

---

### 7.3 Key Design Decisions

#### Init Containers

Wait for MongoDB before starting app:

```bash id="init1"
nc -z <mongodb-host> 27017
```

---

#### Health Probes

* Readiness → receives traffic only when ready
* Liveness → restarts if unhealthy

---

#### Resource Limits

Prevents resource exhaustion per pod.

---

#### Sealed Secrets

Encrypted secrets stored safely in Git.

---

### 7.4 Network Policies (Zero Trust)

```text
Default → DENY ALL

Allowed:
✔ External → Frontend
✔ Frontend → BFF
✔ BFF → Backend
✔ Backend → MongoDB
✔ MongoDB → MongoDB
✔ All → DNS

Blocked:
✘ External → Backend
✘ External → MongoDB
✘ Frontend → MongoDB
```

---

### 7.5 Validate Helm Charts

```bash id="helm2"
helm lint helm/auth
helm template medimesh-auth helm/auth --debug

helm template medimesh-infrastructure helm/infrastructure | grep "kind:"
```

---

## 8. Sealed Secrets Setup

### Why Sealed Secrets?

Kubernetes Secrets are only base64 encoded (NOT secure).
Sealed Secrets encrypt them safely for Git storage.

---

### Flow

```text
Secret → kubeseal → SealedSecret → ArgoCD → Cluster Secret
```

---

### 8.1 Install Controller

```bash id="seal1"
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

kubectl rollout status deployment sealed-secrets-controller -n kube-system
kubectl get pods -n kube-system | grep sealed-secrets
```

---

### 8.2 Install kubeseal CLI

```bash id="seal2"
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz

tar -xvzf kubeseal-0.24.0-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

kubeseal --version
```

---

### 8.3 Create Application Secret

```bash id="seal3"
kubectl create secret generic medimesh-secrets \
  --from-literal=JWT_SECRET="your-secret" \
  --from-literal=ADMIN_USERNAME="admin" \
  --from-literal=ADMIN_PASSWORD="password" \
  --from-literal=MONGO_INITDB_ROOT_USERNAME="mongoroot" \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD="password" \
  --namespace medimesh-backend \
  --dry-run=client -o yaml > medimesh-secrets.yaml

kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  < medimesh-secrets.yaml \
  > medimesh-secrets-sealed.yaml

kubectl apply -f medimesh-secrets-sealed.yaml

kubectl get secret medimesh-secrets -n medimesh-backend

rm medimesh-secrets.yaml
```

---

### 8.4 MongoDB Secret

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

kubectl get secret medimesh-mongo-secret -n medimesh-db
```

---

## 9. Argo Rollouts Setup

### Why Argo Rollouts?

Adds advanced deployment strategies:

* Blue/Green
* Canary

---

### Blue/Green Flow

```text
Blue → current version
Green → new version (no traffic)

Test Green → Promote → Switch traffic
Abort → rollback instantly
```

---

### 9.1 Install Argo Rollouts

```bash id="roll1"
kubectl create namespace argo-rollouts

kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

kubectl rollout status deployment argo-rollouts -n argo-rollouts

kubectl get pods -n argo-rollouts
```

---

### 9.2 Install kubectl Plugin

```bash id="roll2"
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64

sudo install -o root -g root -m 0755 \
  kubectl-argo-rollouts-linux-amd64 \
  /usr/local/bin/kubectl-argo-rollouts

kubectl argo rollouts version
```

---

### 9.3 Dashboard

```bash id="roll3"
kubectl argo rollouts dashboard &
```

Access: http://localhost:3100

---

### 9.4 Manage Rollouts

```bash id="roll4"
kubectl argo rollouts list rollouts -n medimesh-backend

kubectl argo rollouts get rollout medimesh-auth -n medimesh-backend

kubectl argo rollouts promote medimesh-auth -n medimesh-backend

kubectl argo rollouts abort medimesh-auth -n medimesh-backend

kubectl argo rollouts retry rollout medimesh-auth -n medimesh-backend

kubectl argo rollouts restart medimesh-auth -n medimesh-backend
```

---

