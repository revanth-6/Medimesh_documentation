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

