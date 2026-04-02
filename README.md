# Kubernetes Cluster Setup (1 Master + 2 Worker) using containerd

## 📌 Overview
This guide explains how to create a Kubernetes cluster using:
- 1 Master Node (controls the cluster)
- 2 Worker Node (runs applications)
- containerd (container runtime)

---

## 🧠 What are we doing? (Simple Explanation)

We are:
1. Preparing machines (network + system settings)
2. Installing container runtime (containerd)
3. Installing Kubernetes tools
4. Creating cluster (Master)
5. Connecting Worker to Master
6. Enabling networking (so pods can talk)

---

## 🧠 Command Mapping (IMPORTANT)

| Step | Where to Run |
|------|-------------|
| Step 1 | Master + Worker |
| Step 2 | Master + Worker |
| Step 3 | Master + Worker |
| Step 4 | ONLY Master |
| Step 5 | ONLY Master |
| Step 6 | ONLY Worker |
| Step 7 | ONLY Master |
| Step 8 | ONLY Master |

---

# 🟢 Step 1: System Preparation (All Nodes)


sudo swapoff -a
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

❓ Why?
swapoff → Kubernetes does not support swap (performance issue)
br_netfilter → allows network traffic between pods
ip_forward=1 → allows packet forwarding between nodes

👉 Without this → cluster networking will fail

fail

🟢 Step 2: Install containerd (All Nodes)
sudo apt update
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

Edit config:
sudo nano /etc/containerd/config.toml

Change:

SystemdCgroup = true

Restart:

sudo systemctl restart containerd
sudo systemctl enable containerd

❓ Why?
containerd runs containers (pods run inside containers)
Kubernetes NEEDS a container runtime
SystemdCgroup=true → required for kubelet compatibility

👉 Without this → pods will not start

🟢 Step 3: Install Kubernetes Tools (Both Nodes)
sudo apt update
sudo apt install -y kubeadm kubelet kubectl
sudo systemctl enable kubelet
❓ Why?
kubeadm → used to create cluster
kubelet → runs on every node and manages pods
kubectl → CLI to control cluster

👉 Without these → no Kubernetes functionality

🔴 Step 4: Initialize Master
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
❓ Why?
This creates the Kubernetes control plane
Starts components:
API Server
Scheduler
Controller Manager

👉 This is the brain of Kubernetes

🔴 Setup kubectl (Master only)
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
❓ Why?
Allows you to run kubectl commands
Connects CLI to cluster

👉 Without this → kubectl will not work

🔴 Step 5: Get Join Command (Master only)
kubeadm token create --print-join-command
❓ Why?
Generates secure token
Worker uses this to join cluster

🔵 Step 6: Join Worker Node
sudo kubeadm join <MASTER-IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
❓ Why?
Connects worker to master
Registers node in cluster

👉 Without this → worker is not part of cluster

🔴 Step 7: Install Network Plugin (VERY IMPORTANT)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
❓ Why?
Kubernetes does NOT provide networking by default
Flannel:
Assigns IPs to pods
Allows pod-to-pod communication

👉 Without this → nodes stay NotReady

🔴 Step 8: Verify Cluster
kubectl get nodes

Expected Output:
master   Ready
worker-1   Ready
worker-2   Ready
