# Cluster K3S

## Labs Setup
### List Server
```
loadbalancer: 10.10.10.59/24
cp1: 10.10.10.160/24
cp2: 10.10.10.182/24
cp3: 10.10.10.54/24
worker1: 10.10.10.69/24
worker2: 10.10.10.165/24
worker3: 10.10.10.62/24
```

### K3S Component
```
- kubernetes
- containerd
- calico
- coredns
- etcd
- runc
- metric-server
- traefik
- helm-controller
- local-path-provisioner
```

### Topology
![High Availability K3S](k3s-architecture-ha-embedded.svg)

## Pre Install
### Add New User
```
useradd -m -s /bin/bash rafiryd
passwd rafiryd

echo "rafiryd ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers.d/rafiryd
```

### Set hostname
```
// on node loadbalancer
hostnamectl set-hostname loadbalancer

// on node cp1
hostnamectl set-hostname cp1

// on node cp2
hostnamectl set-hostname cp2

// on node cp3
hostnamectl set-hostname cp3

// on node worker1
hostnamectl set-hostname worker1

// on node worker2
hostnamectl set-hostname worker2

// on node worker3
hostnamectl set-hostname worker3
```

### Set Timedate
```
timedatectl set-timezone Asia/Jakarta
```

### Set IP Static
```
vim /etc/netplan/00-installer-config.yaml
```

edit
```
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens160:
      dhcp4: false
      addresses: [10.10.10.x/24]
      routes:
        - to: default
          via: 10.10.10.254
      nameservers:
        addresses: [1.1.1.1,8.8.8.8]
  version: 2
```

### Update & Upgrade
```
apt update -y && apt upgrade -y
```

## Setup Load Balancer
### Install HAProxy
```
apt install haproxy -y
```

### Configure HAProxy
```
vim /etc/haproxy/haproxy.cfg 
```

edit 
```
frontend kubernetes-frontend
    bind *:6443
    mode tcp
    option tcplog
    timeout client 10s
    default_backend kubernetes-backend

backend kubernetes-backend
    timeout connect 10s
    timeout server 10s
    mode tcp
    option tcp-check
    balance roundrobin
    server cp1 10.10.10.160:6443 check fall 3 rise 2
    server cp2 10.10.10.182:6443 check fall 3 rise 2
    server cp3 10.10.10.54:6443 check fall 3 rise 2
```

verify
```
haproxy -c -f /etc/haproxy/haproxy.cfg
```

restart
```
systemctl restart haproxy
```

check listen port
```
ss -tulpn | grep 6443
```

## Set up Cluster
### Setup control plane
Setup Cluster K3S Without Flannel we use Calico as CNI
```
curl -sfL https://get.k3s.io |  INSTALL_K3S_VERSION=v1.27.3+k3s1 sh -s - server --cluster-init --tls-san 10.10.10.59 --flannel-backend=none
```

get token
```
cat /var/lib/rancher/k3s/server/token
```

example token
```
qwertyuiop
```

### Setup other control plane
```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.27.3+k3s1 sh -s - server --server https://10.10.10.59:6443 --token "qwertyuiop" --tls-san 10.10.10.59 --flannel-backend=none
```

### Setup worker node
```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.27.3+k3s1 sh -s - agent --server https://10.10.10.59:6443 --token "qwertyuiop"
```

## Install Addon on cluster 
### Calico
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
```

## Opsional 
### Reset Cluster
on server
```
/usr/local/bin/k3s-uninstall.sh
```
on agent
```
/usr/local/bin/k3s-agent-uninstall.sh
```