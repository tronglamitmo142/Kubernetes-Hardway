# Kubernetes-Hardway
Note about Kubernetes 

# Table of contents 

1. [Theory](#theory)  
2. [Practical](#practical)
   1. [Setup Kubernetes cluster using kubeadm](#kubeadm)

## 2. Practical <a name="practical"></a>
### 2.1. Setup Kubernetes cluster in AWS using kubeadm <a name="kubeadm"></a>

Workflow of Kubeadm: https://devopscube.com/setup-kubernetes-cluster-kubeadm/

Kubeadm Port requirements 

![](./images/Screenshot%202023-05-01%20at%2020.22.27.png)

1. Provisioning ec2 instance
- 1 master node, 2 worker node
- Instance type: t2.medium
- Public IP enable 
2. Update **system's apt package manger index** and update package required to install containerd: 
```bash 
# Update the package index
sudo apt-get update
# Update packages required for HTTPS package repository access
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release
```
3. Allow forwading Ipv4 by loading the br_netfilter module: 
```bash
# Load br_netfilter module
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
- [modprobe](https://linux.die.net/man/8/modprobe): program to add/remove modules from the Linux Kernel.
- [overlay](https://towardsdatascience.com/exploring-the-power-of-overlay-file-systems-in-linux-containers-d846724ec06d#:~:text=In%20the%20context%20of%20Linux,preserving%20the%20original%20image%20intact.): the feature of linux to build overlay file system. 
- br_netfilter: The br_netfilter module is required to enable transparent masquerading and to facilitate Virtual Extensible LAN (VxLAN) traffic for communication between Kubernetes pods across the cluster nodes. We need this to install CRI (Container Runtime interface)

4. Allow Node's iptables to correctly view bridged traffic 
```bash 
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
# Apply sysctl params without reboot
sudo sysctl --system
```

5. Install [containerd](https://github.com/containerd/containerd)

```bash 
# Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# Set up the repository
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# Install containerd
sudo apt-get update
sudo apt-get install -y containerd.io=1.6.15-1
```