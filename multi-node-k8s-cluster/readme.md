Installing k8s using kubeadm

## Kubeadm is a tool designed to bootstrap a full-scale Kubernetes cluster. It takes care of all heavy lifting related to cluster provisioning and automates the process completely.

I am using Ubuntu 20.04 with kubeadm to setup my k8s cluste on AWS.

My proof of concepts contain one control plane node and two worker nodes. I have used t2.medium for all three instances (4GB RAM, 2vCPUs, 30GB memory)

1. Create AWS EC2 instance
Login to the AWS console
Switch to Services -> EC2 -> Launch Instance
Select the ‘Ubuntu’ image and give the Name 
Select the instance type as t2.medium
Create a key-value pair or use existing one
Create a new security group. 
Give 30GB as instance storage
Launch instance

Repeat the steps on all three instances; name them K8s-worker-1, k8s-worker-2, k8s-master. 
Choose the same security group for all three instances. Note: Because of the ease of future steps, I have added one inbound rule with the Type “All Traffic” and Source “Anywhere -IPv4”. However, AWS does not recommend this (For security reasons), and We must add the rules for which required access from outside to the instance

Finally, use EC2 Instance connect to open the terminal in your browser.

Connect to the 3 instances in separate tabs through the browser. It will help you to do the installation at the same time.

2. Install K8s Cluster on Ubuntu 20.

# From Step a to Step d, you must do for all 3 instances

Step a: 
## Update and reboot the server

sudo su
sudo apt update
sudo apt-get update && sudo apt-get upgrade -y
sudo reboot -f

Step b:
## Install, kubelet, kubeadm and kubectl

sudo apt -y install curl apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo “deb https://apt.kubernetes.io/ kubernetes-xenial main” | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
snap install kubeadm --classic
snap install kubectl --classic
snap install kubelet --classic
sudo apt-mark hold kubelet kubeadm kubectl

kubectl version --client && kubeadm version

Step c:
## Disable Firewall and Swap: this is important for communication between nodes and pod-to-pod communications

ufw disable #disable firewall
swapoff -a #disable swap
sudo sed -i '/swap/d' /etc/fstab
sudo mount -a #confirm setting is correct
free -h

Step d:
## Install Container Runtime (Containerd)
## I am using Containerd, other options include Docker, CRI-O

# Configure persistent loading of modules
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

# Load at runtime 
sudo modprobe overlay
sudo modprobe br_netfilter

# Ensure sysctl params are set
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload configs
sudo sysctl --system

# Install required packages
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

# Add Docker repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Install containerd
sudo apt update
sudo apt install -y containerd.io

# Configure containerd and start service
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status containerd

Step e:
## Initialize the master node

# Enable kubelet service
sudo apt-get update && sudo apt-get install -y apt-transport-https
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install kubelet

sudo systemctl enable kubelet

# we ar einitializing the machine that will run th econtrol plane components, API server and etcd (called the cluste db)

# Pull container images
sudo apt-get install cri-tools
which crictl
export PATH=$PATH:/usr/local/bin

sudo kubeadm config images pull --cri-socket unix:///run/containerd/containerd.sock

# Initialize Kubernetes Cluster
kubeadm init --apiserver-advertise-address=INSTANCE_PRIVATE_IP --pod-network-cidr=192.168.0.0/16 --ignore-preflight-errors=all

# To start using your cluster, you need to run the following as a regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Alternatively, if you are the root user, you can run
export KUBECONFIG=/etc/kubernetes/admin.conf

Step f:
# Install Network Plugin on the Master

# Deploy the Calico network
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml
watch kubectl get pods --all-namespaces
