# Kubernetes Cluster Setup Guide

This guide provides step-by-step instructions to set up a Kubernetes cluster on Ubuntu nodes.

## Prerequisites

- Ubuntu installed on all nodes
- Root or sudo access on all nodes

## Steps

### Step 1: Update and Upgrade Ubuntu (all nodes)

```sh
sudo apt update
sudo apt upgrade
```

### Step 2: Disable Swap (all nodes)

```sh
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Step 3: Add Kernel Parameters (all nodes)

Create a configuration file to load necessary kernel modules:

```sh
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure the critical kernel parameters for Kubernetes:

```sh
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

### Step 4: Install Containerd Runtime (all nodes)

Install necessary packages and add Docker repository:

```sh
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

Update the package list and install containerd:

```sh
sudo apt update
sudo apt install -y containerd.io
```

Configure containerd to use systemd as cgroup manager:

```sh
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Step 5: Add Apt Repository for Kubernetes (all nodes)

Install transport packages and add Kubernetes repository:

```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Step 6: Install Kubectl, Kubeadm, and Kubelet (all nodes)

```sh
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Step 7: Initialize Kubernetes Cluster with Kubeadm (master node)

```sh
kubeadm init
```

After the initialization is complete, configure kubectl on the master node:

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 8: Add Worker Nodes to the Cluster (worker nodes)

Use the `kubeadm join` command provided at the end of the `kubeadm init` process on the master node. Example:

```sh
kubeadm join <master-node-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Step 9: Install Kubernetes Network Plugin (master node)

Install Cilium CLI:

```sh
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

Install Cilium on Kubernetes:

```sh
cilium install --version 1.15.5
```

### Step 10: Verify the Cluster and Test (master node)

```sh
kubectl get nodes
kubectl get pods -n kube-system
```

Your Kubernetes cluster should now be up and running. Ensure all nodes are listed, and the necessary pods are running within the `kube-system` namespace.
