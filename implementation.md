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
