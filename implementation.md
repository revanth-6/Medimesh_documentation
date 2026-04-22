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

MediMesh runs on a self-managed Kubernetes cluster built using `kubeadm`. The cluster has one master (control plane) node and two worker nodes. A separate NFS server provides persistent storage for MongoDB.

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

---

## 10. ArgoCD Setup

### Why ArgoCD?

ArgoCD implements GitOps.

Git = single source of truth
Cluster state always matches Git

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
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=Ready pods \
  --all -n argocd \
  --timeout=300s

kubectl get pods -n argocd
```

---

### 10.2 Access UI

```bash id="argo2"
kubectl port-forward svc/argocd-server \
  -n argocd 8080:443 &
```

Access: https://localhost:8080

---

### 10.3 Get Admin Password

```bash id="argo3"
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

---

### 10.4 Install CLI

```bash id="argo4"
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
```

Login:

```bash id="argo5"
argocd login localhost:8080 \
  --username admin \
  --password <password> \
  --insecure
```

---

### 10.5 Add Repo

```bash id="argo6"
argocd repo add https://github.com/Medimesh-grp3/manifest_repo \
  --username <username> \
  --password <pat>
```

---

### 10.6 Create Applications

```bash id="argo7"
argocd app create medimesh-infrastructure \
  --repo https://github.com/Medimesh-grp3/manifest_repo \
  --path helm/infrastructure \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace medimesh-backend \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

---

### Backend Services Loop

```bash id="argo8"
for service in auth user doctor appointment vitals \
               pharmacy ambulance complaint forum bff; do
  argocd app create medimesh-${service} \
    --repo https://github.com/Medimesh-grp3/manifest_repo \
    --path helm/${service} \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace medimesh-backend \
    --sync-policy automated \
    --auto-prune \
    --self-heal
done
```

---

### 10.7 Verify

```bash id="argo9"
argocd app list
argocd app get medimesh-auth
argocd app sync medimesh-auth
```

---

## 11. kGateway Setup

### Why kGateway?

Single entry point using Gateway API.

---

### Routing

```text id="gw1"
 /auth        → auth service
 /user        → user service
 /doctor      → doctor service
 /api         → BFF
 /            → frontend
```

---

### 11.1 Install Gateway CRDs

```bash id="gw2"
kubectl apply -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

kubectl get crd | grep gateway
```

---

### 11.2 Install kGateway

```bash id="gw3"
helm repo add kgateway-dev https://kgateway-dev.github.io/kgateway/
helm repo update

helm install kgateway kgateway-dev/kgateway \
  --namespace kgateway-system \
  --create-namespace \
  --version 2.0.0
```

---

### 11.3 Verify Gateway

```bash id="gw4"
kubectl get gateway -n medimesh-frontend
kubectl get httproute -n medimesh-frontend
```

---

## 12. Helm Deployment

### Deployment Order

```text id="order1"
1. NFS
2. MongoDB
3. Infrastructure
4. Backend
5. Frontend
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
helm install medimesh-mongo helm/mongo \
  -n medimesh-db

kubectl get pods -n medimesh-db -w
kubectl get pvc -n medimesh-db
```

---

### 12.3 Infrastructure

```bash id="helm5"
helm install medimesh-infrastructure helm/infrastructure \
  -n medimesh-backend \
  --create-namespace

kubectl get ns
kubectl get networkpolicy -n medimesh-backend
kubectl get secret medimesh-secrets -n medimesh-backend
```

---

### 12.4 Backend Services

```bash id="helm6"
for service in auth user doctor appointment vitals \
               pharmacy ambulance complaint forum bff; do
  helm install medimesh-${service} helm/${service} \
    -n medimesh-backend
done

kubectl get pods -n medimesh-backend -w
```

---

### 12.5 Frontend

```bash id="helm7"
helm install medimesh-frontend helm/frontend \
  -n medimesh-frontend

kubectl get pods -n medimesh-frontend
kubectl get svc -n medimesh-frontend
kubectl get gateway -n medimesh-frontend
```

---

### 12.6 Verification

```bash id="helm8"
kubectl get pods -n medimesh-frontend
kubectl get pods -n medimesh-backend
kubectl get pods -n medimesh-db
kubectl get pvc -n medimesh-db
```

---

### 12.7 Helm Upgrade

```bash id="helm9"
helm upgrade medimesh-auth helm/auth -n medimesh-backend
helm history medimesh-auth -n medimesh-backend
helm rollback medimesh-auth 1 -n medimesh-backend
```

---


---

## 13. Prometheus & Grafana Setup

### Why Prometheus + Grafana?

* Prometheus → collects metrics
* Grafana → visualizes metrics

---

### 13.1 Add Helm Repo

```bash id="mon1"
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
```

---

### 13.2 Install Monitoring Stack

```bash id="mon2"
kubectl create namespace monitoring

helm install prometheus \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=7d \
  --set grafana.adminPassword=admin123 \
  --set grafana.service.type=NodePort \
  --set prometheus.service.type=NodePort

kubectl rollout status deployment prometheus-grafana -n monitoring

kubectl get pods -n monitoring
```

---

### 13.3 Access Grafana

```bash id="mon3"
kubectl port-forward svc/prometheus-grafana \
  -n monitoring 3000:80 &
```

Access: http://localhost:3000
Login: admin / admin123

---

### 13.4 Access Prometheus

```bash id="mon4"
kubectl port-forward \
  svc/prometheus-kube-prometheus-prometheus \
  -n monitoring 9090:9090 &
```

Access: http://localhost:9090

---

### 13.5 ServiceMonitor

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
```

---

### 13.6 Grafana Dashboards

```text id="mon6"
1860  → Node Exporter
6417  → Cluster Overview
13770 → Full Monitoring
15661 → Argo Rollouts
```

---

### 13.7 Prometheus Queries

```bash id="mon7"
rate(container_cpu_usage_seconds_total{namespace="medimesh-backend"}[5m])

container_memory_usage_bytes{namespace="medimesh-backend"}

kube_pod_container_status_restarts_total{namespace="medimesh-backend"}
```

---

## 14. SonarQube Setup

### Why SonarQube?

Static Application Security Testing (SAST)

Detects:

* Bugs
* Code smells
* Security issues

---

### 14.1 PostgreSQL Setup

```bash id="sonar1"
sudo apt update
sudo apt install -y postgresql postgresql-contrib

sudo systemctl start postgresql
sudo systemctl enable postgresql
```

Create DB:

```bash id="sonar2"
sudo -u postgres psql << EOF
CREATE USER sonar WITH ENCRYPTED PASSWORD 'sonarpassword';
CREATE DATABASE sonarqube OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;
\q
EOF
```

---

### 14.2 Install Java

```bash id="sonar3"
sudo apt install -y openjdk-17-jdk
java -version
```

---

### 14.3 Install SonarQube

```bash id="sonar4"
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.3.0.82913.zip

sudo apt install -y unzip
unzip sonarqube-10.3.0.82913.zip
sudo mv sonarqube-10.3.0.82913 /opt/sonarqube

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
EOF
```

---

### 14.5 Systemd Service

```bash id="sonar6"
sudo tee /etc/systemd/system/sonarqube.service << EOF
[Unit]
Description=SonarQube service
After=network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonar
Group=sonar
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start sonarqube
sudo systemctl enable sonarqube
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
```

---

## 15. GitHub Actions CI Setup

### CI Flow

```text id="ci1"
Push → SonarQube → Build → Snyk → Docker → Trivy → Push → Approval → CD
```

---

### Email Notifications

Sent using SMTP via GitHub Actions:

* Failure
* Approval required
* Success

---

### 15.1 Organization Secrets

```text id="ci2"
SONARQUBE_TOKEN
SONARQUBE_URL
SNYK_TOKEN
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
MANIFEST_REPO_PAT
SMTP_USERNAME
SMTP_PASSWORD
ALERT_EMAIL
```

---

### 15.2 Environment Setup

Create environment: `Prod`

* Required reviewers
* Approval required before deploy

---

### 15.3 Template Repo

```bash id="ci3"
cd template-ci
ls .github/workflows/
```

Expected templates:

```text id="ci4"
template-sonarqube.yml
template-build-snyk.yml
template-docker.yml
template-approval-gate.yml
template-cd.yml
```

---

### 15.4 Service Config Mapping

```text id="ci5"
auth       → values_key: auth
user       → values_key: user
doctor     → values_key: doctor
appointment→ values_key: appointment
```

---

### 15.5 Versioning Strategy

```text id="ci6"
feature:   → minor bump
breaking:  → major bump
default:   → patch bump
```

---

### 15.6 Docker Tags

```text id="ci7"
v1.0.3
v1.0.3-<sha>
latest
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
git commit -m "feat: trigger CI"
git push origin main
```

---


---

## 16. GitHub Actions CD Setup

### CD Flow (GitOps)

```text id="cd1"
CI Success + Approval
        ↓
Update manifest_repo (values.yaml)
        ↓
ArgoCD detects change
        ↓
Applies Helm update
        ↓
Argo Rollouts (Blue/Green)
```

---

### 16.1 Verify Manifest Update

```bash id="cd2"
cd manifest_repo
git pull origin main

git log --oneline -5

cat helm/auth/values.yaml | grep tag
```

---

### 16.2 Verify ArgoCD Sync

```bash id="cd3"
argocd app get medimesh-auth

argocd app sync medimesh-auth
```

---

### 16.3 Complete Deployment

```bash id="cd4"
kubectl get pods -n medimesh-backend

kubectl argo rollouts get rollout medimesh-auth \
  -n medimesh-backend

kubectl port-forward svc/medimesh-auth-svc-preview \
  -n medimesh-backend 5001:5001 &

curl http://localhost:5001/health

kubectl argo rollouts promote medimesh-auth \
  -n medimesh-backend
```

---

### 16.4 Git Retry Logic

```bash id="cd5"
for i in 1 2 3 4 5; do
  git push && break
  git pull --rebase origin main
  sleep $((i * 2))
done
```

---

## 17. OIDC Setup for Vitals

### Why OIDC?

No static credentials.
Short-lived secure authentication.

---

### OIDC Flow

```text id="oidc1"
GitHub → OIDC Token → K8s API → RBAC → Deploy → Token expires
```

---

### 17.1 Configure API Server

Edit:

```bash id="oidc2"
/etc/kubernetes/manifests/kube-apiserver.yaml
```

Add:

```text id="oidc3"
--oidc-issuer-url=https://token.actions.githubusercontent.com
--oidc-client-id=kubernetes
--oidc-username-claim=sub
--oidc-username-prefix=github:
--oidc-groups-claim=repository
```

---

### 17.2 Apply RBAC

```bash id="oidc4"
kubectl apply -f manifest_repo/rbac/github-actions-vitals-rbac.yaml
```

---

### 17.3 Get CA Cert

```bash id="oidc5"
kubectl config view --raw --minify \
  --output 'jsonpath={.clusters[0].cluster.certificate-authority-data}'
```

---

### 17.4 Get API Server

```bash id="oidc6"
kubectl config view --minify \
  --output 'jsonpath={.clusters[0].cluster.server}'
```

---

### 17.5 Self-Hosted Runner

```bash id="oidc7"
mkdir actions-runner && cd actions-runner

curl -o runner.tar.gz -L <runner-url>
tar xzf runner.tar.gz

./config.sh --url <repo-url> --token <token>

sudo ./svc.sh install
sudo ./svc.sh start
```

---

### 17.6 Repo Secrets

```text id="oidc8"
K8S_CA_CERT
K8S_API_SERVER
```

---

### 17.7 Promote Vitals

```bash id="oidc9"
kubectl argo rollouts get rollout medimesh-vitals \
  -n medimesh-backend

kubectl argo rollouts promote medimesh-vitals \
  -n medimesh-backend
```

---

## 18. Full Pipeline Verification

### 18.1 End-to-End Test

```bash id="verify1"
git commit -m "feat: test pipeline"
git push origin main
```

Then verify:

* GitHub Actions
* SonarQube
* Docker Hub
* ArgoCD
* Rollouts

---

### 18.2 Health Checks

```bash id="verify2"
kubectl get pods -n medimesh-frontend
kubectl get pods -n medimesh-backend
kubectl get pods -n medimesh-db
```

---

### Gateway Testing

```bash id="verify3"
curl http://<gateway-ip>/auth/health
curl http://<gateway-ip>/
```

---

### 18.3 Monitoring

```bash id="verify4"
kubectl get pods -n monitoring

kubectl port-forward svc/prometheus-grafana \
  -n monitoring 3000:80 &
```

---

### 18.4 Full System Status

```bash id="verify5"
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get pvc -n medimesh-db
kubectl get hpa -n medimesh-frontend
kubectl get networkpolicy --all-namespaces
```

---

## 19. Troubleshooting Reference

### 19.1 CrashLoopBackOff

```bash id="tr1"
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
```

---

### 19.2 PVC Pending

```bash id="tr2"
kubectl get pvc
kubectl describe pvc <pvc>
kubectl get sc
```

---

### 19.3 ArgoCD Issues

```bash id="tr3"
argocd app get medimesh-auth
argocd app sync medimesh-auth
```

---

### 19.4 CI Failures

Check:

* GitHub Actions logs
* Secrets configuration
* Tokens

---

### 19.5 Sealed Secrets Issues

```bash id="tr4"
kubectl get pods -n kube-system | grep sealed-secrets
kubectl logs -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

---

### 19.6 Rollout Stuck

```bash id="tr5"
kubectl argo rollouts get rollout medimesh-auth \
  -n medimesh-backend

kubectl argo rollouts promote medimesh-auth
```

---

### 19.7 Network Issues

```bash id="tr6"
kubectl exec -it <pod> -- sh

nslookup service-name
wget -qO- http://service:port/health
```

---

### 19.8 SonarQube Issues

```bash id="tr7"
sudo systemctl status sonarqube
curl http://<ip>:9000/api/system/status
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



