# Kubernetes Cluster Setup with kubeadm

This guide explains how to set up a Kubernetes cluster using `kubeadm` on two EC2 instances:

- 1 Master node
- 1 Worker node

## Prerequisites

- 2 EC2 instances running Ubuntu
- Root or sudo access on both machines
- Internet access for downloading packages
- Proper network configuration and security groups

## 1) Run on BOTH EC2 instances (Master + Worker)

### 1.0 Update system and set hostname

```bash
sudo apt update
sudo hostnamectl set-hostname master   # or worker
```

### 1.1 Disable swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

> Required because `kubelet` fails if swap is enabled.

### 1.2 Enable required kernel modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### 1.3 Enable networking for Kubernetes

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

---

## 2) Install Container Runtime (`containerd`)

```bash
sudo apt update
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 3) Install Kubernetes components

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

> These three packages are mandatory on all nodes.

---

## 4) Setup MASTER node

### 4.1 Initialize the cluster

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

This command initializes the Kubernetes control plane.

### 4.2 Configure `kubectl`

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4.3 Install a CNI network plugin

Example: Calico

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

> Required so pods can communicate with each other.

---

## 5) Join the WORKER node to the cluster

After running `kubeadm init`, you will get a join command similar to:

```bash
sudo kubeadm join <MASTER-IP>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

Run this command on the worker node.

---

## 6) Verify the cluster (on the master)

```bash
kubectl get nodes
```

Expected output:

```bash
NAME       STATUS   ROLES           AGE   VERSION
master     Ready    control-plane   XXm   v1.xx
worker     Ready    <none>          XXm   v1.xx
```

---

## 7) Enable kubectl auto-completion

```bash
sudo apt install -y bash-completion

echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "alias k=kubectl" >> ~/.bashrc
echo "complete -F __start_kubectl k" >> ~/.bashrc
source ~/.bashrc
```

---

## Summary

This setup creates a Kubernetes cluster with:

- 1 control-plane node (master)
- 1 worker node
- CNI networking enabled for pod communication
- `kubectl` configured for cluster management

