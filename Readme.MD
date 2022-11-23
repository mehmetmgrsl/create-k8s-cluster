## Creating a K8s Cluster

### 1. Creating 2 VMs (virtual machines) using Vagrant

In this example a Kubernetes cluster will consist of one master and one worker node. For this, we will be using Vagrant and VirtualBox to quickly provision virtual machines.

1.1 Create a dedicated SSH key pair to connect to the VMs:

```ssh-keygen -b 4096```

```vagrant_key``` is used as a key name. 

1.2 Use the created SSH public key in Vagrantfile. (Update the path as well based on your public key path.) 

1.3 Run the following command on the root folder of the project to create 2 VMs (master and worker)
    
```cd  vagrant-provision```

```vagrant up```

- The hostnames and IP addresses of the VMs are below:
  - k8s-master    : 192.168.56.15
  - worker1       : 192.168.56.16

To connect the VMs use the following commands:

k8s-master node:

```ssh vagrant@192.168.56.15 -i $HOME/.ssh/vagrant_key```

worker1 node:

```ssh vagrant@192.168.56.16 -i $HOME/.ssh/vagrant_key```


### 2. Common setup for the VMs

- **Install and configure prerequisites - Letting iptables see bridged traffic**
 
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
```

#### 2.1. Installing the container runtime (containerd)

 ```
 sudo apt-get update
 sudo apt-get install -y containerd
 sudo mkdir -p /etc/containerd
 sudo containerd config default | sudo tee /etc/containerd/config.toml
 sudo systemctl restart containerd
 ```

#### 2.2. Installing kubeadm, kubelet and kubectl

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
# sudo apt-get install -y kubelet=1.24.3-00 kubectl=1.24.3-00 kubeadm=1.24.3-00
sudo apt-mark hold kubelet kubeadm kubectl
```

* The default user and password is ```vagrant/vagrant```

#### 2.4. We need to disable swap in order for the kubelet to work well

```
sudo swapoff -a # disable swap
```

```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab # disable it completely.
```

### 3. Master node setup 

#### 3.1. Initialize the master

```
sudo kubeadm init --kubernetes-version=v1.23.0 --apiserver-advertise-address=192.168.56.15 --pod-network-cidr=192.168.0.0/16
# sudo kubeadm init --kubernetes-version=v1.24.3 --apiserver-advertise-address=192.168.56.15 --pod-network-cidr=192.168.0.0/16 --ignore-preflight-errors=all  --cri-socket /run/containerd/containerd.sock
```

set the KUBECONFIG env variable like below:
```
cp /etc/kubernetes/admin.conf $HOME/
chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
```

#### 3.2. Install Calico

```
#Install the Tigera Calico operator and custom resource definitions.
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml

#Install Calico by creating the necessary custom resource. For more information on configuration options available in this manifest, see the installation reference.
kubectl create -f https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml
```

### 4. Worker node setup 

At the end of the 'kubeadm init' command you can find a needed command like below to join the cluster:
```
sudo kubeadm join --token <TOKEN> <MASTER-IP>:<MASTER-PORT> --discovery-token-ca-cert-hash sha256:<hash>
```

Note: You can recreate a token and print a join command in the MASTER node with running the below command:
```
kubeadm token create --print-join-command
```

### 5. Access kubernetes cluster in your local machine:

- 5.1 Access master node like below:

```ssh vagrant@192.168.56.15 -i $HOME/.ssh/vagrant_key```

- 5.2 Copy the content of the /etc/kubernetes/admin.conf file

- 5.3 Create a conf file on your local machine and paste the content of the previous step into it.

i.e -> $HOME/.kube/config.yaml

- 5.4 Set the KUBECONFIG environment variable, i.e like below:

```export KUBECONFIG=$HOME/.kube/config.yaml```

- 5.5 Run the kubectl get nodes to check whether you can access the cluster or not:

```kubectl get nodes```

Output should be like below:

```
NAME         STATUS   ROLES                  AGE   VERSION
k8s-master   Ready    control-plane,master   58m   v1.23.0
worker1      Ready    <none>                 46m   v1.23.0
```


Resources:
1. https://kosyfrances.com/kubernetes-cluster/
2. https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
3. https://docs.docker.com/engine/install/ubuntu/
4. https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart
5. https://sysnet4admin.gitbook.io/k8s/trouble-shooting/cluster-build/kubelet-is-not-properly-working-on-1.22-version
6. https://www.nocentino.com/posts/2021-12-27-installing-and-configuring-containerd-as-a-kubernetes-container-runtime/
7. https://computingforgeeks.com/deploy-kubernetes-cluster-on-ubuntu-with-kubeadm/
