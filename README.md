# Kubernetes-Hardway
Note about Kubernetes 

# Table of contents 

- [Kubernetes-Hardway](#kubernetes-hardway)
- [Table of contents](#table-of-contents)
  - [1. Theory](#1-theory)
    - [1.1. Networking ](#11-networking-)
  - [2. Practical ](#2-practical-)
    - [2.1. Setup Kubernetes cluster in AWS using kubeadm ](#21-setup-kubernetes-cluster-in-aws-using-kubeadm-)

## 1. Theory
### 1.1. Networking <a name="networking"></a>
- Basic networking in Linux system: [networking](./networking.md)
- [CNI](#cni)  
- [Service Networking](#service)
- [DNS](#dns)
1. [CNI](https://github.com/containernetworking/cni)<a name="cni"></a>
- **Standard** that define how program should develop to solving container runtime.
- Docker doesn't follow CNI 
- Network addons list: https://kubernetes.io/docs/concepts/cluster-administration/addons/
- Deploy weave network: https://v1-22.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#steps-for-the-first-control-plane-node  

1. Pod Networking<a name="pod-networking"></a>  
- Pod networking model:
  - Every Pod should have IP Address 
  - Every Pod should be able to communicate with other pod in the same node
  - Every Pod should be able to communicate with every other pod in other nodes without NAT 

- CNI help us to solve this problem automatically 
![](images/Screenshot%202023-05-02%20at%2007.29.02.png)
- CNI configuration file is part of `kubelet`.
```bash
--cni-conf-dir=/etc/cni/net.d
--cni-bin-dir=/etc/cni/bin
```
- View kubelet options
```bash
ps -aux | grep kubelet
```

```bash
ls /opt/cni/bin
```

```bash
ls /etc/cni/net.d
```

- CNI Weave Work:
  - IP range: 10.32.0.1 -> 10.47.255.254
```bash 
# Deploy weave work as a deamon set
```

- IP Address Management (IPAM)
  - CNI manage IP assigment for pod, they have 2 plugins:
    - DHCP
    - host-local
    - Can define in `ipam` spec in network configuration file

3. Service Networking<a name="service"></a>
![](/images/Screenshot%202023-05-02%20at%2009.09.05.png)
- Virtual object in cluster. When `service` is created, `kube-proxy` create a forwarding rules that tell every trafic to the service ip will go to the pod's ip. The rules is defined in iptables by default. 
- We can specify range of servers (example clusterip):
```bash
kube-api-server --service-cluster-ip-range ipNet
```

```bash
ps aux | grep kube-api-server
```
- Inspect rules in `iptables`:
```bash
iptables -L -t nat | grep services 
```
```bash
cat /var/log/kube-proxy.log
```
- ClusterIP
- NodePort

4. DNS<a name="dns"></a>
- Service resolution: 
![](images/Screenshot%202023-05-02%20at%2009.24.59.png)
- Pod resolution: 
![](images/Screenshot%202023-05-02%20at%2009.26.01.png)
- It's recomendation to use [CoreDNS](https://github.com/coredns/coredns)
- When CoreDNS is created, it also create a service type clusterIP. The ip of services is defined by kubelet
```bash
cat /var/lib/kubelet/config.yaml
```

5. Ingress<a name="ingress"></a>
- Act as layer 7 Load Balancer in kubernetes cluster 
- Ingress = Ingress controller (deploy) + Ingress resource (configure)
- Conponent of ingress controller 
![](./images/Screenshot%202023-05-02%20at%2010.44.48.png)
- Example of ingress rule 
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: critical-space
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /pay
        backend:
          serviceName: pay-service
          servicePort: 8282
```
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

6. Configure the systemd cgroup driver

```bash 
# Configure the systemd cgroup driver
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

7. Install kubeadm, kubectl, kubelet: 

```bash
# Add the Google Cloud packages GPG key
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
# Add the Kubernetes release repository
sudo add-apt-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
# Update the package index to include the Kubernetes repository
sudo apt-get update
# Install the packages
sudo apt-get install -y kubeadm=1.24.3-00 kubelet=1.24.3-00 kubectl=1.24.3-00
```

8. Prevent automatic update

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

9. Initialize the control-plane node

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --kubernetes-version=stable-1.24
```

We can print the `join` command
```bash
kubeadm token create --print-join-command
``` 

10. Initialize user's default kubectl configuration using the admin kubeconfig file generated by kubeadm

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

11. Create the Calio network plugin for pod networking 
```bash
kubectl apply -f https://clouda-labs-assets.s3.us-west-2.amazonaws.com/k8s-common/1.24/scripts/calico.yaml
```

12. Verify 

```bash
kubectl get nodes
```

13. Join another nodes into cluster. That nodes need to have kubectl, kubeadm, kubelet

```bash 
sudo kubeadm join 10.0.0.100:6443 --token ... --discovery-token-ca-cert-hash sha256:...
```

