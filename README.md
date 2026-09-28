# Kubernates-Cluster-Setup
Kubernetes Cluster Setup using kubeadm (2 EC2 Instances)
This guide explains how to set up a Kubernetes cluster using kubeadm on:

1 Master Node
1 Worker Node

⚙️ 1. Run on BOTH EC2 instances (Master + Worker)
⚙️ 1. Run on BOTH EC2 instances (Master + Worker)
1.0 Update and Set Hostname
sudo apt update 
sudo hostnamectl set-hostname master/worker 
1.1 Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
Required because kubelet fails if swap is enabled.

1.2 Enable kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
1.3 Enable networking for Kubernetes
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
⚙️ 2. Install Container Runtime (containerd)
sudo apt update
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
⚙️ 3. Install Kubernetes components (OFFICIAL METHOD)
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
These 3 are mandatory on all nodes

🚀 4. Setup MASTER node
4.1 Initialize cluster
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
kubeadm initializes control plane

4.2 Configure kubectl (IMPORTANT)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
4.3 Install CNI (Network Plugin)
Example: Calico

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
Required so pods can communicate

🔗 5. Join WORKER node to cluster
After running kubeadm init, you will get a join command like:

kubeadm join <MASTER-IP>:6443 --token <token> \
--discovery-token-ca-cert-hash sha256:<hash>
Run this command on the Worker Node using sudo.

✅ 6. Verify cluster (on Master)
kubectl get nodes
Expected output:

NAME       STATUS   ROLES           AGE   VERSION
master     Ready    control-plane   XXm   v1.xx
worker     Ready    <none>          XXm   v1.xx
✅ 7. For Auto-Completion Execute below commands
sudo apt install -y bash-completion
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "alias k=kubectl" >> ~/.bashrc
echo "complete -F __start_kubectl k" >> ~/.bashrc
source ~/.bashrc
