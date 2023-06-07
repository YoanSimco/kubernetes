# kubernetes

## Table of content

<!--ts-->
- [kubernetes](#kubernetes)
  - [Table of content](#table-of-content)
  - [🚀 Install Kubernetes cluster using Kubeadm and Cilium CNI (Debian based)](#-install-kubernetes-cluster-using-kubeadm-and-cilium-cni-debian-based)
    - [Disable swap](#disable-swap)
    - [Install container runtime (containerd)](#install-container-runtime-containerd)
      - [Prerequisites - Forwarding IPv4 and letting iptables see bridged traffic](#prerequisites---forwarding-ipv4-and-letting-iptables-see-bridged-traffic)
      - [Install containerd](#install-containerd)
      - [Configure containerd (use systemd cgroup driver)](#configure-containerd-use-systemd-cgroup-driver)
    - [Install kubeadm, kubelet and kubectl](#install-kubeadm-kubelet-and-kubectl)
    - [Create the cluster with Kubeadm](#create-the-cluster-with-kubeadm)
      - [Enable pods scheduling on the control plane nodes (optional)](#enable-pods-scheduling-on-the-control-plane-nodes-optional)
    - [Install Cilium CNI](#install-cilium-cni)
      - [Install Cilium CLI](#install-cilium-cli)
      - [Install Cilium](#install-cilium)
      - [Validate Cilium installation](#validate-cilium-installation)
  - [⬆️ Upgrade Kubernetes cluster installed using Kubeadm and Cilium CNI](#️-upgrade-kubernetes-cluster-installed-using-kubeadm-and-cilium-cni)
    - [Determine which version to upgrade to](#determine-which-version-to-upgrade-to)
    - [Upgrade Kubeadm](#upgrade-kubeadm)
      - [Verify the upgrade plan](#verify-the-upgrade-plan)
      - [Choose a version to upgrade to](#choose-a-version-to-upgrade-to)
    - [Upgrade Cilium CNI](#upgrade-cilium-cni)
    - [Upgrade the node](#upgrade-the-node)
      - [Drain the node](#drain-the-node)
      - [Upgrade kubelet and kubectl](#upgrade-kubelet-and-kubectl)
      - [Upgrade container runtime (containerd)](#upgrade-container-runtime-containerd)
      - [Uncordon the node](#uncordon-the-node)
    - [Verify the status of the cluster](#verify-the-status-of-the-cluster)
  - [🧹 Uninstall Kubernetes cluster installed using Kubeadm](#-uninstall-kubernetes-cluster-installed-using-kubeadm)
    - [Reset Kubeadm](#reset-kubeadm)
    - [Remove CNI configuration](#remove-cni-configuration)
    - [Remove Kubernetes configuration](#remove-kubernetes-configuration)
    - [Reset iptables rules](#reset-iptables-rules)
    - [Clear containerd containers and images](#clear-containerd-containers-and-images)

<!-- Created by https://github.com/ekalinin/github-markdown-toc -->
<!-- Added by: yoan, at: Wed Jun  7 09:10:15 CEST 2023 -->

<!--te-->

## 🚀 Install Kubernetes cluster using Kubeadm and Cilium CNI (Debian based)

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### Disable swap

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin

```bash
sudo swapoff -a
sudo rm /swap.img
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Install container runtime (containerd)

https://kubernetes.io/docs/setup/production-environment/container-runtimes/

#### Prerequisites - Forwarding IPv4 and letting iptables see bridged traffic

https://kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic

```bash
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

#### Install containerd

https://docs.docker.com/engine/install/ubuntu/#installation-methods

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Add Docker’s official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Set up Docker's apt repository
echo \
"deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
"$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y containerd.io
sudo apt-mark hold containerd.io
```

https://github.com/containerd/containerd/blob/main/docs/getting-started.md#customizing-containerd

```bash
sudo containerd config default > /etc/containerd/config.toml
```

#### Configure containerd (use systemd cgroup driver)

https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd

In `/etc/containerd/config.toml`:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

```bash
sudo systemctl restart containerd
```

### Install kubeadm, kubelet and kubectl

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl

# Download the Google Cloud public signing key
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

# Add the Kubernetes apt repository
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubeadm kubelet kubectl
sudo apt-mark hold kubeadm kubelet kubectl
```

### Create the cluster with Kubeadm

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Enable pods scheduling on the control plane nodes (optional)

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Install Cilium CNI

https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/

#### Install Cilium CLI

https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

#### Install Cilium

https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-cilium

```bash
cilium install
```

#### Validate Cilium installation

https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#validate-the-installation

```bash
cilium status --wait
kubectl get nodes
```

## ⬆️ Upgrade Kubernetes cluster installed using Kubeadm and Cilium CNI

https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

### Determine which version to upgrade to

https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#determine-which-version-to-upgrade-to

```bash
sudo apt update
sudo apt-cache madison kubeadm
```

### Upgrade Kubeadm

https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#call-kubeadm-upgrade

```bash
sudo apt-mark unhold kubeadm && \
sudo apt update && sudo apt install -y kubeadm=1.27.x-00 && \
sudo apt-mark hold kubeadm
```

#### Verify the upgrade plan

```bash
sudo kubeadm upgrade plan
```

#### Choose a version to upgrade to

```bash
sudo kubeadm upgrade apply v1.27.x
```

### Upgrade Cilium CNI

See https://docs.cilium.io/en/stable/operations/upgrade/

### Upgrade the node

#### Drain the node

https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#drain-the-node

```bash
kubectl drain <node-to-drain> --ignore-daemonsets
```

#### Upgrade kubelet and kubectl

https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-kubelet-and-kubectl

```bash
sudo apt-mark unhold kubelet kubectl && \
sudo apt update && sudo apt install -y kubelet=1.27.x-00 kubectl=1.27.x-00 && \
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

#### Upgrade container runtime (containerd)

```bash
sudo apt-mark unhold containerd.io && \
sudo apt update && sudo apt install -y containerd.io && \
sudo apt-mark hold containerd.io

sudo systemctl restart containerd.io
```

#### Uncordon the node

https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#uncordon-the-node

```bash
kubectl uncordon <node-to-uncordon>
```

### Verify the status of the cluster

```bash
kubectl get nodes
```

## 🧹 Uninstall Kubernetes cluster installed using Kubeadm

### Reset Kubeadm

https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/

```bash
sudo kubeadm reset
```

### Remove CNI configuration

```bash
sudo rm -rf /etc/cni/net.d
```

### Remove Kubernetes configuration

```bash
rm -rf ~/.kube/*
```

### Reset iptables rules

```bash
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -F
sudo iptables -X

sudo ip6tables -P INPUT ACCEPT
sudo ip6tables -P FORWARD ACCEPT
sudo ip6tables -P OUTPUT ACCEPT
sudo ip6tables -t nat -F
sudo ip6tables -t mangle -F
sudo ip6tables -F
sudo ip6tables -X
```

### Clear containerd containers and images

```bash
sudo ctr -n k8s.io c rm $(sudo ctr -n k8s.io c ls -q)
sudo ctr -n k8s.io i rm $(sudo ctr -n k8s.io i ls -q)
```
