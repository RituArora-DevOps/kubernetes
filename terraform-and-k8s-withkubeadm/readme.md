We plan to set up the following infrastructure, leveraging Terraform for efficient provisioning. This approach is designed to optimize lab time, ensuring a swift infrastructure deployment and allowing you to concentrate on the subsequent cluster installation tasks without undue delays.

This setup mirrors the structure found in CKA Ultimate Mocks and the actual exam format. You initiate your tasks on a distinct node (student-node) and utilize SSH to connect to the cluster nodes. It's noteworthy that SSH connections can only be initiated following the direction of the arrows; for example, you cannot SSH directly from the controlplane to node01. Exiting to student-node is a prerequisite, aligning with the exam's approach where student-node assumes the role of a bastion host.

To facilitate seamless browsing of any NodePort services you create, we'll establish a direct connection from your workstation to the node ports of the worker nodes. Some basic security configurations will be implemented, including restricting API Server access to only the student-node, enabling SSH access exclusively from the student-node to the cluster nodes, and setting up security groups for required Kubernetes and Weave CNI ports.

Security considerations limit the suitability of this configuration for a production cluster. Ideally, kube nodes should be in private subnets, behind a NAT gateway or in an airgapped environment. Access to the API server and etcd should be more tightly controlled. The use of default VPC is discouraged. Additionally, caution is advised since node ports will be exposed globally, and a cloud load balancer coupled with an ingress controller is recommended for secure ingress provision.

The Terraform code will also handle the following configurations:

Assign host names to the nodes: controlplane, node01, node02.
Set up the content of /etc/hosts on all nodes for easy use of SSH commands from student-node.
Generate and distribute a key pair for SSH-based instance login.

## Install terraform

# From the cloudshell prompt

curl -O https://releases.hashicorp.com/terraform/1.6.2/terraform_1.6.2_linux_amd64.zip
unzip terraform_1.6.2_linux_amd64.zip
mkdir -p ~/bin
mv terraform ~/bin/
terraform version

# Clone the repo 
git clone https://github.com/RituArora-DevOps/kubernetes

cd terraform

-----

## Provision the infrastructure using Terraform

terraform init 
terraform plan
terraform apply

# Copy the output obtained from last command to a notepad. 
Outputs:

address_node01 = "44.202.187.178"
address_node02 = "35.173.138.222"
address_student_node = "aws_instance.student_node.public_ip"
connect_student_node = "ssh ubuntu@54.81.115.50"

----

# Once all the instances are in running state ssh into student-node.
# Here student-node can be considered as bastian host.

ssh ubuntu@54.81.115.50

## Prepare the student node

# Install latest version of kubectl and place in the user programs directory

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin

kubectl version

---

# Note: It should amongst other things tell you
# The connection to the server localhost:8080 was refused - did you specify the right host or port?
# which is fine, since we haven't installed kubernetes yet.

## Configure OS, container runtime and kube packeges

# be logged into student-node, then ssh into controlplane, node01 and node02 one by one.
ssh controlplane

# There's no step to disble swap, since EC2 instances are by default with swap disabled

sudo -i         # become root

apt-get update   # update and install required packages to use the K8s apt repo 
apt-get install -y apt-transport-https ca-certificates curl

# set up the required kernel modules and make them persistent 

cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# Set the required kernel parameters and make them persistent

cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

# install the container runtime
apt-get install -y containerd

## Set up the container runtime to utilize systemd Cgroups. This step is often overlooked by many , and neglecting it can lead to a functional control plane but with all pods entering a crash-loop state. Additionally, executing kubectl commands may result in an error such as:
# The connection to the server x.x.x.x:6443 was refused - did you specify the right host or port?

# create default configuration
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# Edit the configuration to set up CGroups # Scroll down till you find a line with SystemdCgroup = false. Edit it to be SystemdCgroup = true
vi /etc/containerd/config.toml

# Restart containerd
systemctl restart containerd

# Get latest version of Kubernetes and store in a shell variable

KUBE_LATEST=$(curl -L -s https://dl.k8s.io/release/stable.txt | awk 'BEGIN { FS="." } { printf "%s.%s", $1, $2 }') 

# download k8s public signing key
mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/${KUBE_LATEST}/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# add the k8s apt repo
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${KUBE_LATEST}/deb/ /" > /etc/apt/sources.list.d/kubernetes.list

# install kubelet, kubeadm, kubectl
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl     # to pin the version of kubelet kubeadm kubectl

# Configure crictl in case we need it to examine running containers

crictl config \
    --set runtime-endpoint=unix:///run/containerd/containerd.sock \
    --set image-endpoint=unix:///run/containerd/containerd.sock

exit         # exit root shell
exit         # exit controlplane and return to student-node

# repeat the above steps for node01 and node02

----
## Boot up controlplane

ssh controlplane
sudo -i
kubeadm init     #boot teh control plane and copy the printed join command for later use on the worker nodes

# Install network plugin (weave)
kubectl --kubeconfig /etc/kubernetes/admin.conf apply -f "https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s-1.11.yaml"

# check the cluster is up and running 
kubectl --kubeconfig /etc/kubernetes/admin.conf get pods -n kube-system

exit    # exit root shell

# prepare kubeconfig file and copy it to student-node 
{
  sudo cp /etc/kubernetes/admin.conf .
  sudo chmod 666 admin.conf
}

exit    # exit to student node

# Copy kubeconfig down from controlplane to student-node and set proper permissions

mkdir -p ~/.kube
scp controlplane:~/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config

# test it 
kubectl get pods -n kube-system

-----

## join the worker nodes

ssh node01
sudo -i 
# paste the join command that was saved earlier
### example: kubeadm join 172.31.23.173:6443 --token 1iqu0d.3ycafbdt6q4ih2vl \
        --discovery-token-ca-cert-hash sha256:c63fe4466a55d1bbad7863e116c38afba695c828a42fa436eb2e2325bb677503
###

exit
exit

# repeat the steps for node02

# back on student-node, check all nodes are up and running

-----

## create a test service

kubectl run nginx --image nginx --expose --port 80    # deploy and expose an nginx pod

# edit the spec: part of the service type to NodePort
# test from your browser
http://<public-IP-node01-or-02>:30080