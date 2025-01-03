 __  __           __                                            __                       
/\ \/\ \         /\ \                                          /\ \__                 
\ \ \/'/'  __  __\ \ \____     __   _ __    ___      __    ____\ \ ,_\    __    ____  
 \ \ , <  /\ \/\ \\ \ '__`\  /'__`\/\`'__\/' _ `\  /'__`\ /',__\\ \ \/  /'__`\ /',__\ 
  \ \ \\`\\ \ \_\ \\ \ \L\ \/\  __/\ \ \/ /\ \/\ \/\  __//\__, `\\ \ \_/\  __//\__, `\
   \ \_\ \_\ \____/ \ \_,__/\ \____\\ \_\ \ \_\ \_\ \____\/\____/ \ \__\ \____\/\____/
    \/_/\/_/\/___/   \/___/  \/____/ \/_/  \/_/\/_/\/____/\/___/   \/__/\/____/\/___/
      ___             _           _                         
     |  _|___ ___ _ _| |_ ___ _ _| |___ ___  
     |  _| .'|  _| | | '_| . | | | | -_|  _|
WWW .|_| |__,|_| |___|_,_|_  |___|_|___|_|.COM
Name: Kubernetes Cluster Installation Script
Author: faruk guler
Date: 2025

#Server Inventory [Hosts]
Kubectl:  192.168.44.140
Master:   192.168.44.145
Worker1:  192.168.44.146
Worker2:  192.168.44.147
Worker3:  192.168.44.148

Docs:
https://kubernetes.io/
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
https://labs.play-with-k8s.com/

#Hosts file:
Master1 node: sudo hostnamectl set-hostname master
Node1 worker: sudo hostnamectl set-hostname node1
Node2 worker: sudo hostnamectl set-hostname node2
Node3 worker: sudo hostnamectl set-hostname node3

#DNS(Domain Name System) Integration:
127.0.0.1       localhost
192.168.44.145  master
192.168.44.146  worker1
192.168.44.147  worker2

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

#Uniq Servers Verify:
lsb_release -a
ip a
sudo cat /sys/class/dmi/id/product_uuid

#Firewall Ports and Protocols:
>> Control plane:
TCP 6443 (Inbound): Kubernetes API server – All
TCP 2379-2380 (Inbound): etcd server client API – kube-apiserver, etcd
TCP 10250 (Inbound): Kubelet API – Self, Control plane
TCP 10259 (Inbound): kube-scheduler – Self
TCP 10257 (Inbound): kube-controller-manager – Self
$ sudo ss -tuln | grep 6443

>> Worker node(s):
TCP 10250 (Inbound): Kubelet API – Self, Control plane
TCP 10256 (Inbound): kube-proxy – Self, Load balancers
TCP 30000-32767 (Inbound): NodePort Services – All
$ sudo ss -tuln | grep 10250

#SELINUX
$ sudo nano /etc/selinux/config
SELINUX=disabled
$ sudo reboot
$ sestatus

#Swap Areas:
$ cat /proc/swaps
$ swapon --show
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
$ free -m
$ lscpu

---------Installing-------------->>

#Kernel and Network modules activate:
$ sudo modprobe overlay
$ sudo modprobe br_netfilter

$ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

#System Apply:
$ sudo sysctl --system

#Runtime Containerd:
$ sudo apt update
$ sudo apt install containerd
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now containerd
$ sudo systemctl start containerd
$ sudo mkdir -p /etc/containerd
$ containerd config default | tee /etc/containerd/config.toml
$ sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
$ sudo systemctl restart containerd

#Install Kubeadm:
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
#apt-mark hold kubelet kubeadm kubectl
#apt-mark unhold kubelet kubeadm kubectl
$ nc 127.0.0.1 6443 -v
$ journalctl -u kubelet
$ journalctl -xfe

#Kubernetes Cluster:
$ sudo kubeadm config images pull
$ sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<ip> --control-plane-endpoint=<ip>
$ sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.44.145 --control-plane-endpoint=192.168.44.145

#Kubernetes Nodes Configuration:
sudo scp /etc/kubernetes/admin.conf root@192.168.44.148:/etc/kubernetes/admin.conf
/etc/kubernetes/admin.conf
~/.kube/config

#Kubectl:
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl config
$ kubectl config get-contexts

#Networking: Calico:
$ #kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
$ #kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
$ #kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml

#Taint:
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-
$ kubectl taint nodes --all node-role.kubernetes.io/master-

#Kubectl Auto-Completion:
$ source <(kubectl completion bash)
$ echo "source <(kubectl completion bash)" >> ~/.bashrc
#kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
#sudo apt-get install bash-completion

#Installing Helm:
https://helm.sh/docs/intro/install/
https://artifacthub.io/packages/search?ts_query_web=rancher

#Kubernetes dashboard:
#Rancher
#Headlamp

#Begin >>>
$ kubectl version
$ kubectl cluster-info
$ kubectl get nodes
$ kubernetes get nodes -owide
$ kubectl get cs
$ kubectl get all
$ kubectl get all -A