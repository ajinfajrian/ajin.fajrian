---
author: "Fajrian"
title: "High Availability Kubernetes Control Plane with Kube-VIP BGP"
date: "2023-12-13"
tags: [
    "kubernetes",
    "containerd",
    "kube-vip",
    "high availability",
]
toc: true
## disable draft, if finished
draft: true
---

Hello folks. Just finished my playground lab a few days ago, and it's been long time not write some posts to internet. \
There is 2 options to get high availability control plane with kube-vip, first is with arp mode (layer 2), and the second is with bgp peering (layer 3). \
The reason im build this cluster with kube-vip bgp mode is i just want to learn about bgp routing can advertising a virtual ip address :).


> Instance Specification
| Hostname | CPU   | Memory  | IP Address | Description |
| -------- | ----- | ------- | ----------- | ----------- |
| jin-bgp-router   | 1 vCPU  | 2 GB    | 10.20.31.51   | |
| jin-k8s-master-1 | 4 vCPU  | 4 GB    | 10.20.31.11   | |
| jin-k8s-master-2 | 4 vCPU  | 4 GB    | 10.20.31.12   | |
| jin-k8s-master-3 | 4 vCPU  | 4 GB    | 10.20.31.13   | |
| jin-k8s-worker-1 | 4 vCPU  | 8 GB    | 10.20.31.21   | |
| jin-apiserver    |         |         | 30.30.30.10   | Virtual IP |

> Software Specification
- OS                 : Ubuntu 22.04 LTS
- Bird BGP Version   : v2.0.8
- Kubernetes Version : v1.28.4
- Containerd Version : v1.6.26
- Kube-VIP           : v0.4.0

## Topology

- 1 bird bgp instance
- 3 control plane nodes
- 1 worker nodes

```
                                                                                           
                                                                             BAREMETAL     
    +----------------------------------------------------------------------------------+   
    |                                                                                  |   
    |                                                                                  |   
    |                              +--------------------+                              |   
    |                              |   jin-bgp-router   |                              |   
    |                +-------------|    10.20.31.51     |--------------+               |   
    |                |             +--------------------+              |               |   
    |                |                        |     bgp routes         |               |   
    |     asn: 65000 |             asn: 65000 |    advertisement:      |asn: 65000     |   
    |                |                        |    30.30.30.0/24       |               |   
    |                |                        |                        |               |   
    |     +--------------------+   +--------------------+   +--------------------+     |   
    |     |  jin-k8s-master-1  |   |  jin-k8s-master-2  |   |  jin-k8s-master-3  |     |   
    |     |  ens3:10.20.31.11  |   |  ens3:10.20.31.12  |   |  ens3:10.20.31.13  |     |   
    |     |  lo  :30.30.30.10  |   |  lo  :30.30.30.10  |   |  lo  :30.30.30.10  |     |   
    |     +--------------------+   +--------------------+   +--------------------+     |   
    |                |                        |                        |               |   
    |                |                        |                        |               |   
    |                |             +--------------------+              |               |   
    |                |             |    jin-apiserver   |              |               |   
    |                +-------------|     30.30.30.10    |--------------+               |   
    |                              +--------------------+                              |   
    |                                         |                                        |   
    |                                         |                                        |   
    |                                         |                                        |   
    |                              +--------------------+                              |   
    |                              |  jin-k8s-worker-1  |                              |   
    |                              |  ens3:10.20.31.21  |                              |   
    |                              +--------------------+                              |   
    |                                                                                  |   
    +----------------------------------------------------------------------------------+   
```

we will bind advertising ip address from bgp to **interface lo** on all control plane as we don't want multiple devices that have the same address on public interfaces. [reference](https://github.com/kube-vip/kube-vip/blob/main/docs/hybrid/static/index.md#bgp).

## Setup BGP

installing bird bgp on instance `jin-bgp-router`
```sh
sudo apt update && sudo apt install -y bird2
```
and now we start edit bird config on `/etc/bird/bird.conf`
```sh
log syslog all;

router id 10.20.31.51; ## ip interface of bgp-router

protocol device {
}

protocol direct {
  ipv4;
}

protocol kernel {
  learn on;
  persist;
  merge paths on;
  ipv6 { export all; };
}

protocol static {
  ipv4;
}

protocol static my_routes {
  ipv4;
  route 30.30.30.0/24 via 10.20.31.51;
}

filter export_my_routes {
 if proto = "my_routes" then {
  accept;
 }
 reject;
}

protocol bgp master1 {
  local as 65000;
  neighbor 10.20.31.11 as 65000;

  ipv4 {
    import all;
    export filter export_my_routes;
  };
}

protocol bgp master2 {
  local as 65000;
  neighbor 10.20.31.12 as 65000;

  ipv4 {
    import all;
    export filter export_my_routes;
  };
}

protocol bgp master3 {
  local as 65000;
  neighbor 10.20.31.13 as 65000;

  ipv4 {
    import all;
    export filter export_my_routes;
  };
}
```

short explanation on bird config: \
- **protocol static my_routes**: we can set route virtual ip pool whatever we want. and in this part i prefer choose 30.30.30.0/24.
- **protocol bgp master1-3**: its bgp peer client, i set static pod on master node and advertise vip for master node only, so we don't have to put worker node on bird config.

restart bird service to apply new configuration.
```sh
sudo systemctl restart bird
```
that's it and we've done configure the bgp server, now we installing kubernetes depedencies.

## Setup Depedencies on all nodes
> add persistent overlay and br_netfilter module
```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
> enable overlay and br_netfilter module
```sh
sudo modprobe overlay
sudo modprobe br_netfilter
```
> add sysctl for iptables to watch traffic bridge
```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
> apply sysctl params without reboot instances
```sh
sudo sysctl --system
```

mapping hostname to ip addresses
```sh
cat <<EOF | sudo tee -a /etc/hosts
30.30.30.10 jin-apiserver
10.20.31.11 jin-k8s-master-1
10.20.31.12 jin-k8s-master-2
10.20.31.13 jin-k8s-master-3

10.20.31.21 jin-k8s-worker-1
EOF
```

#### Import repository
**import docker / containerd repository**
```sh
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

**import kubernetes repository**
```sh
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

#### Installing
installing containerd as cri and kubernetes with specific version below, and prevent package being upgrade.
```sh
sudo apt update && sudo apt install -y kubelet=1.28.4-00 kubeadm=1.28.4-00 kubectl=1.28.4-00 containerd.io=1.6.26-0
sudo apt-mark hold kubelet kubeadm kubectl containerd.io
```

after all package installation is complete, we will configure containerd with default config and set containerd to use systemd cgroups.
```sh
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml && sed -i "s/SystemdCgroup = false/SystemdCgroup = true/g" /etc/containerd/config.toml
sudo systemctl restart containerd
``` 

## Setup kube-vip
on jin-k8s-master-1, create container to run kube-vip.
```sh
## change variable K8S_MASTER with another control plane ip address
export K8S_MASTER=10.20.30.11
export VIP=30.30.30.10
export INTERFACE=lo
export IMAGE_VIP=v0.4.0

alias kube-vip="sudo ctr image pull ghcr.io/kube-vip/kube-vip:$IMAGE_VIP; sudo ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$IMAGE_VIP vip /kube-vip"

## static pod
kube-vip manifest pod \
                  --interface $INTERFACE \
                  --address $VIP \
                  --controlplane \
                  --bgp \
                  --peerAddress 10.20.31.51 \
                  --peerAS 65000 \
                  --localAS 65000 \
                  --bgpRouterID $K8S_MASTER | sudo tee /etc/kubernetes/manifests/kube-vip.yaml              
```

after we create static pods on jin-k8s-master-1, execute these command on `jin-k8s-master-2` & `jin-k8s-master-3` as well. everything is same except `variable K8S_MASTER`. change it manually with our ip address. \
that script will generate static pod. if you ping the $VIP it will not responding, we have to bootstraping / run `kubeadm init` then wait to connect to VIP. it's because kubelet will parse and execute all manifests, including `kube-vip`. [reference](https://github.com/kube-vip/kube-vip/blob/main/docs/hybrid/index.md#static-pods)

## Bootstraping Kubernetes


### Verify

## Conclusion

## References
- [Say good-bye to HAProxy and Keepalived with kube-vip on your HA K8s control plane](https://inductor.medium.com/say-good-bye-to-haproxy-and-keepalived-with-kube-vip-on-your-ha-k8s-control-plane-bb7237eca9fc)
- [Cluster High Availability Kubernetes](https://blog.syslog.my.id/posts/kubernetes-ha-part1/)
- [Kube-vip as a Static Pod](https://github.com/kube-vip/kube-vip/blob/main/docs/hybrid/static/index.md#bgp)