# Kubernetes Cluster Setup (1 Master + 2 Worker) using containerd

##  Overview
This guide explains how to create a Kubernetes cluster using:
- 1 Master Node (controls the cluster)
- 2 Worker Nodes (run applications)
- containerd (container runtime)

---

##  What are we doing? (Simple Explanation)

We are:
1. Preparing machines (network + system settings)
2. Installing container runtime (containerd)
3. Installing Kubernetes tools
4. Creating cluster (Master)
5. Connecting Workers to Master
6. Enabling networking (so pods can talk)

---

##  Command Mapping (IMPORTANT)

| Step | Where to Run |
|------|-------------|
| Step 1 | Master + Workers |
| Step 2 | Master + Workers |
| Step 3 | Master + Workers |
| Step 4 | ONLY Master |
| Step 5 | ONLY Master |
| Step 6 | ONLY Workers |
| Step 7 | ONLY Master |
| Step 8 | ONLY Master |

---

#  Step 1: System Preparation (All Nodes)
```bash
sudo swapoff -a
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```
 Why?
- swapoff → Kubernetes does not support swap
- br_netfilter → enables pod network traffic
- ip_forward=1 → allows communication between nodes

 Without this → cluster networking will fail

#  Step 2: Install containerd (All Nodes)
```
sudo apt update
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```
Edit config:
```bash
sudo nano /etc/containerd/config.toml
```
Change:
```
SystemdCgroup = true
```
Restart:
```
bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```
 Why?
- containerd runs containers
- Kubernetes needs a runtime
- SystemdCgroup=true → required for kubelet

 Without this → pods will not start

#  Step 3: Install Kubernetes Tools (All Nodes)
```bash
sudo apt update
sudo apt install -y kubeadm kubelet kubectl
sudo systemctl enable kubelet
```
Why?
- kubeadm → create cluster
- kubelet → runs pods
- kubectl → manage cluster

#  Step 4: Initialize Master
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
 Why?
- Creates control plane
- Starts API Server, Scheduler, Controller

 This is the brain of Kubernetes

#  Setup kubectl (Master only)
```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
 Why?
- Enables kubectl to talk to cluster

#  Step 5: Get Join Command (Master only)
```bash
kubeadm token create --print-join-command
```
 Why?
- Generates secure token for workers

#  Step 6: Join Worker Nodes

Run this on worker-1 and worker-2:
```bash
sudo kubeadm join <MASTER-IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```
 Why?
- Connects workers to master
- Registers nodes in cluster

#  Step 7: Install Network Plugin (VERY IMPORTANT)
```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
 Why?
- Kubernetes has no default networking
- Flannel:
- Assigns IPs to pods
- Enables pod communication

Without this → nodes stay NotReady

#  Step 8: Verify Cluster
```bash
kubectl get nodes
```

#  Expected Output
- master     Ready
- worker-1   Ready
- worker-2   Ready
