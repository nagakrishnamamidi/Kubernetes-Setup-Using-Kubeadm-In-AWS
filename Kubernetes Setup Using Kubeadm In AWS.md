Setup a Multi Node Kubernetes Cluster Using Kubeadm

#### What is kubeadm
`kubeadm` is a tool to bootstrap the Kubernetes cluster, which installs all the control plane components and gets the cluster ready for you.

Along with control plane components(ApiServer, ETCD, Controller Manager, Scheduler) , it helps with the installation of various CLI tools such as Kubeadm, Kubelet and kubectl

### Ways to install Kubernetes

![image](https://github.com/user-attachments/assets/5391de53-36bc-4574-81cf-4b1fefbda9e3)

>Note: In this demo, we will be installing Kubernetes on VMs on cloud(Self-Managed)

### Steps to set up the Kubernetes cluster

![image](https://github.com/user-attachments/assets/e0943ad5-2d13-4128-8147-1ef644c62955)


>Note: If using a Mac Silicon chip, Multipass is the recommended choice as Virtualbox has some compatibility issues or you can spin up virtual machines on the cloud and use that


**If you are using AWS EC2 servers, you need to allow specific traffic on specific ports as below**

1) Provision 3 VMs, 1 Master, and 2 Worker nodes.
2) Create 2 security groups, attach 1 to the master and the other to two worker nodes using the below details.
https://kubernetes.io/docs/reference/networking/ports-and-protocols/

![image](https://github.com/user-attachments/assets/58d66bcb-ed00-453f-99b2-8df9f4393cac)


4) If you are using AWS EC2, you need to disable **source destination check** for the VMs using the [doc](https://docs.aws.amazon.com/vpc/latest/userguide/work-with-nat-instances.html#EIP_Disable_SrcDestCheck)

### Run the below steps on the Master VM
1) SSH into the Master EC2 server

2)  Disable Swap using the below commands
```bash
swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
3) Forwarding IPv4 and letting iptables see bridged traffic

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Verify that the br_netfilter, overlay modules are loaded by running the following commands:
lsmod | grep br_netfilter
lsmod | grep overlay

# Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

4) Install container runtime

```
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Check that containerd service is up and running
systemctl status containerd
```

5) Install runc

```
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

6) install cni plugin

```
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
```

7) Install kubeadm, kubelet and kubectl

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.29.6-1.1 kubeadm=1.29.6-1.1 kubectl=1.29.6-1.1 --allow-downgrades --allow-change-held-packages
sudo apt-mark hold kubelet kubeadm kubectl

kubeadm version
kubelet --version
kubectl version --client
```
>Note: The reason we are installing 1.29, so that in one of the later task, we can upgrade the cluster to 1.30


===========In Master Node Start====================
# Steps Only For Kubernetes Master

# Switch to the root user.

sudo su -

# Initialize Kubernates master by executing below commond.

sudo kubeadm init --apiserver-advertise-address Private IPv4 VM  --pod-network-cidr 10.244.0.0/16  --cri-socket  unix:///var/run/containerd/containerd.sock


#exit root user & exeucte as normal user

exit

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


# To verify, if kubectl is working or not, run the following command.

kubectl get pods -o wide --all-namespaces

#You will notice from the previous command, that all the pods are running except one: ‘kube-dns’. For resolving this we will install a # pod network. To install the weave pod network, run the following command:

kubectl apply -f https://reweave.azurewebsites.net/k8s/v1.29/net.yaml

kubectl get nodes

kubectl get pods --all-namespaces

# Get token

kubeadm token create --print-join-command

=========In Master Node End====================


Add Worker Machines to Kubernates Master
=========================================

Copy kubeadm join token from and execute in Worker Nodes to join to cluster



kubectl commonds has to be executed in master machine.

Check Nodes
=============

kubectl get nodes


