## Creating a K8s Cluster

### 1. Creating 3 VMs (virtual machines) using Vagrant
- There will be 3 VMs. One for the master node and others for the worker nodes.

- Run the following command on the root folder of the project:

    ```vagrant up```

- The hostnames and IP addresses of the VMs are below:
  - k8s-master  : 192.168.56.80
  - k8s-worker1 : 192.168.56.81
  - k8s-worker2 : 192.168.56.82

### 2. Common setup for the VMs

#### 2.1. Installing the container runtime (containerd)


- **Install and configure prerequisites**
 
```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

- **Setup Docker Repository and Install Docker Engine on Ubuntu**

1. Update the apt package index and install packages to allow apt to use a repository over HTTPS:

 ```sudo apt-get update```

 ```
 sudo apt-get install \
 ca-certificates \
 curl \
 gnupg \
 lsb-release
 ```

2. Add Docker’s official GPG key:

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

3. Use the following command to set up the stable repository. To add the nightly or test repository
```
 echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

4. Update the apt package index, and install the latest version of Docker Engine and containerd
```
 sudo apt-get update
 sudo apt-get install docker-ce docker-ce-cli containerd.io
```


#### 2.2. Installing kubeadm, kubelet and kubectl

- Letting iptables see bridged traffic 
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

- Update the apt package index and install packages needed to use the Kubernetes apt repository:
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

- Download the Google Cloud public signing key:
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

- Add the Kubernetes apt repository:
```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

- Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
sudo apt-get install -y kubelet=1.23.0-00 kubectl=1.23.0-00 kubeadm=1.23.0-00
sudo apt-mark hold kubelet kubeadm kubectl
```

#### 2.3. We need to disable swap in order for the kubelet to work well

```
sudo swapoff -a # disable swap
```

```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab # disable it completely.
```

### 3. Master setup 

#### 3.1. Initialize the master

```
#Change your docker driver

touch /etc/docker/daemon.json
cat <<EOF | sudo tee /etc/docker/daemon.json
{
 "exec-opts": ["native.cgroupdriver=systemd"],
 "log-driver": "json-file",
 "log-opts": {
   "max-size": "100m"
 },
 "storage-driver": "overlay2"
}
EOF

systemctl daemon-reload && systemctl restart docker
```

```
sudo kubeadm init --apiserver-advertise-address=192.168.56.80 --pod-network-cidr=192.168.0.0/16
```

#### 3.2. Install Calico

```
#Install the Tigera Calico operator and custom resource definitions.
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml

#Install Calico by creating the necessary custom resource. For more information on configuration options available in this manifest, see the installation reference.
kubectl create -f https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml
```




Resources:
1. https://kosyfrances.com/kubernetes-cluster/
2. https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
3. https://docs.docker.com/engine/install/ubuntu/
4. https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart
5. https://sysnet4admin.gitbook.io/k8s/trouble-shooting/cluster-build/kubelet-is-not-properly-working-on-1.22-version