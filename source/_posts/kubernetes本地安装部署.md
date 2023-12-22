---
title: kubernetes本地安装部署
date: 2023-12-22 09:00:40
author: 古可
tags: kubernetes
categories: DevOps
---

## 本地环境

1. MacOS M1
2. 虚拟机: Paralles Desktop
3. 虚拟机管理工具: vagrant

## vagrant 配置如下

```js
NUM_WORKER_NODES=2
IP_NW="10.1.0."
IP_START=10

ENV['VAGRANT_NO_PARALLEL'] = 'yes'

Vagrant.configure("2") do |config|
    config.vm.provision "shell", inline: <<-SHELL
        apt-get update -y
        echo "$IP_NW$((IP_START))  master-node" >> /etc/hosts
        echo "$IP_NW$((IP_START+1))  worker-node01" >> /etc/hosts
        echo "$IP_NW$((IP_START+2))  worker-node02" >> /etc/hosts
    SHELL
    config.vm.box = "jharoian3/ubuntu-22.04-arm64"
    config.vm.box_check_update = true

    config.vm.define "master" do |master|
      master.vm.hostname = "master-node"
      master.vm.network "private_network", ip: IP_NW + "#{IP_START}"
      master.vm.provider "parallels" do |vb|
          vb.memory = 3072
          vb.cpus = 3
      end
    end


    (1..NUM_WORKER_NODES).each do |i|
      config.vm.define "node0#{i}" do |node|
        node.vm.hostname = "worker-node0#{i}"
        node.vm.network "private_network", ip: IP_NW + "#{IP_START + i}"
        node.vm.provider "parallels" do |vb|
            vb.memory = 1536
            vb.cpus = 2
        end
      end
    end
  end

```

> 配置中特别需要注意的点就是需要用到的镜像，需要使用arm架构的镜像，并且需要支持parallels。我这里使用的是jharoian3/ubuntu-22.04-arm64，你也可以去vagrant的镜像库去找一找。

## 需要在虚拟机中执行的脚本

这里没有直接把sh加入到Vagrant的配置里，是因为我想一句一句去执行以下shell中的命令。因为个人环境千差万别，指不定就在哪一行出错了，以这样的方式能快速定位问题，解决问题

1. common.sh

```shell
#! /bin/bash

KUBERNETES_VERSION="1.28.4-00"

# Make sure Ubuntu has the lates updates
sudo apt update
sudo apt -y full-upgrade

# Add the Google Kubernetes repo
# sudo apt -y install curl apt-transport-https gnupg2 ca-certificates
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install the kubelet, kubectl and kubeadm
sudo apt update
sudo apt -y install kubelet="$KUBERNETES_VERSION" kubectl="$KUBERNETES_VERSION" kubeadm="$KUBERNETES_VERSION"
sudo apt-mark hold kubelet kubeadm kubectl

# Get rid of the swap
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
sudo mount -a
free -h

# Configure the needed modules and persist them
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

# Ensure sysctl params are set
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

# Add Docker repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Install containerd
sudo apt update
sudo apt install -y containerd.io

# Configure containerd and start the service
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml

# Start both containerd and kubelet
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl enable kubelet

```
2. master.sh



```shell
#! /bin/bash

MASTER_IP="10.1.0.10"
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"
KUBERNETES_VERSION="1.28.4"
CILIUM_VERSION="1.12.3"

# Pull the kubernetes images
sudo kubeadm config images pull 

# Init Kubernetes
sudo kubeadm init \
--skip-phases=addon/kube-proxy \
--apiserver-advertise-address=$MASTER_IP  \
--apiserver-cert-extra-sans=$MASTER_IP \
--pod-network-cidr=$POD_CIDR \
--node-name $NODENAME

# KUBECONFIG for in-VM kubectl usage
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Untaint the control node to be used as a worker too
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl taint nodes --all  node-role.kubernetes.io/control-plane-

# Install helm
sudo snap install helm --classic

# Install Cilium wiht its helm chart
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
--version $CILIUM_VERSION \
--namespace kube-system \
--set kubeProxyReplacement=strict \
--set k8sServiceHost=$MASTER_IP \
--set k8sServicePort=6443   \
--set hubble.listenAddress=":4244" \
--set hubble.relay.enabled=true \
--set hubble.ui.enabled=true  

# Generete KUBECONFIG on the host
sudo cp -f /etc/kubernetes/admin.conf /vagrant/configs/config
# sudo touch /vagrant/configs/join.sh
# sudo chmod +x /vagrant/configs/join.sh       

# Generete the kubeadm join command on the host
kubeadm token create --print-join-command | sudo tee /vagrant/configs/join.sh

```

3. node.sh

node 本质上只需要Join进集群就行，config可以不用

```shell
#! /bin/bash

NODENAME=$(hostname -s)

# Join the control node
/bin/bash /vagrant/configs/join.sh -v

# KUBECONFIG for in-VM kubectl usage
sudo -i -u vagrant bash << EOF
mkdir -p /home/vagrant/.kube
sudo cp -i /vagrant/configs/config /home/vagrant/.kube/
sudo chown 1000:1000 /home/vagrant/.kube/config
kubectl label node $(hostname -s) node-role.kubernetes.io/worker=worker-new
EOF

```