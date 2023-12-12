---
author: "Fajrian"
title: "[K8s] Kubernetes High Availability Control Plane"
date: "2023-12-12"
tags: [
    "kubernetes",
    "containerd",
    "keepalived",
]
toc: true
---

A highly available Kubernetes cluster ensures your applications run without outages which is required for production. In this connection, there are plenty of ways for you to choose from to achieve high availability. 

This post demonstrates how to configure Keepalived and HAproxy for load balancing and achieve high availability.

### Lab Environment
#### Hardware Specification
|    Hostname  |     IP         | Specification | Description |
| ------------ | -------------  | ------------- | ------------- |
| jin-vip | 10.20.31.10 |      | Virtual IP |
| jin-master-1 | 10.20.31.11 |   4Gb/4vcpu   | Master |
| jin-master-2 | 10.20.31.12 |   4Gb/4vcpu   | Master |
| jin-master-3 | 10.20.31.13 |   4Gb/4vcpu   | Master |
| jin-worker-1 | 10.20.31.21 |   8Gb/4vcpu   | Worker |


### Pre-Installation
##### Setup Proxy Client (Optional)
Configure a proxy client for containerd. This is intended for those who operate in environments with restricted internet access and are only permitted to access the internet via a proxy server. 
```sh
## set proxy on containerd

$ mkdir -p /etc/systemd/system/containerd.service.d
$ cat <<EOF | sudo tee /etc/systemd/system/containerd.service.d/http-proxy.conf 
[Service]
Environment="no_proxy=127.0.0.1,172.28.216.125"
Environment="http_proxy=http://172.28.216.125:3128"
Environment="https_proxy=http://172.28.216.125:3128"
EOF
```
##### Disable swap
If you are using Kubernetes with version [v1.22 or below]([url](https://kubernetes.io/blog/2021/08/09/run-nodes-with-swap-alpha/)), it is necessary to disable your swap memory for the kubelet to function properly.
```sh
## disable swap on all master & worker node

$ sudo sed -i '/ swap / s/^/#/' /etc/fstab
$ sudo swapoff -a
```
##### Kernel config
```sh
$ echo "net.ipv4.ip_nonlocal_bind=1" | sudo tee /etc/sysctl.d/ip_nonlocal_bind.conf

$ cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF


$ cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

$ sudo modprobe overlay
$ sudo modprobe br_netfilter

$ sudo sysctl --system
```

##### Mapping hosts (execute on all master and workers)
Enhance the readability of your host mapping by adding the following lines:
```sh
cat <<EOF | sudo tee -a /etc/hosts
## k8s-cluster
10.20.31.10 jin-vip
10.20.31.11 jin-master-1
10.20.31.12 jin-master-2
10.20.31.13 jin-master-3
10.20.31.21 jin-worker-1
EOF
```
### Containerd
##### Install containerd
Installing containerd on **all master & workers**
```sh
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/containerd.list
$ sudo apt update -y && sudo apt upgrade -y && sudo apt install -y containerd.io
```
##### Configuring containerd
```sh
$ sudo mkdir -p /etc/containerd
$ containerd config default | sudo tee /etc/containerd/config.toml
$ sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
$ sudo systemctl daemon-reload && sudo systemctl restart containerd
```

The `crictl.yaml` file is used to configure settings for `crictl`, enabling users to customize its behavior and connect to different container runtimes. Typically, this configuration file is located at `/etc/crictl.yaml`.
```sh
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 30
debug: false
EOF
```

```sh
$ crictl  ps -a
CONTAINER     IMAGE       CREATED       STATE         NAME           ATTEMPT  
```
##### Download kubernetes package (master & worker)
```sh
$ sudo apt-get update && sudo apt-get install -y apt-transport-https vim git curl wget
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
$ sudo apt update -y && sudo apt -y install kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
```
### Create Virtual IP Address
Setup Virtual IP with Keepalived, we gonna use this vip as a high availability control plane
```sh
## install keepalive on all master node
sudo apt update && sudo apt install -y keepalived 
```
- create keepalived config on master 1
```sh
sudo vi /etc/keepalived/keepalived.conf

### start from here
vrrp_script chk_api {
  script "/usr/bin/pgrep containerd"
  interval 1
  timeout 1
}

vrrp_instance kube-api {
  state MASTER
  interface ens3 ## public interface
  advert_int 1
  priority 103
  virtual_router_id 51
  authentication {
      auth_type PASS
      auth_pass P4$$w0rd
  }
  unicast_src_ip 10.20.31.11
  unicast_peer {
    10.20.31.12
    10.20.31.13
  }
  virtual_ipaddress {
      10.20.31.10/24
  }
  track_script {
      chk_api
  }
}
```

```sh
sudo systemctl restart keepalived
```

for temporary we set pgrep to `containerd` until the kube-apiserver is going up. and now we verify the interface, make sure the vip was already up

```sh
$ ip a show ens3
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    link/ether 52:54:00:6e:e2:f8 brd ff:ff:ff:ff:ff:ff
    altname enp0s3
    inet 10.20.31.11/24 brd 10.20.31.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet 10.20.31.10/24 scope global secondary ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe6e:e2f8/64 scope link
       valid_lft forever preferred_lft forever

$ ping jin-vip -c 2
PING jin-vip (10.20.31.10) 56(84) bytes of data.
64 bytes from jin-vip (10.20.31.10): icmp_seq=1 ttl=64 time=0.017 ms
64 bytes from jin-vip (10.20.31.10): icmp_seq=2 ttl=64 time=0.035 ms
```
### Bootstraping cluster
Bootstraping kubernetes cluster from master-1 with virtual ip we configured before.
```yaml
$ vi kubeadm-init.yaml

---
apiServer:apiServer:
  certSANs:
  - 127.0.0.1
  - localhost
  - 10.20.31.10 # kube-vip 
  extraArgs:
    authorization-mode: Node,RBAC
    enable-aggregator-routing: "true"
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
controlPlaneEndpoint: jin-vip:6443 ## keepalived vip
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: stable
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.94.0.0/16
---
```

Bootstraping kubernetes cluster jin-master-1
```sh
$ sudo kubeadm init --config=kubeadm-init.yaml --skip-phases=addon/kube-proxy --upload-certs

...
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
...
```

##### Accessing cluster
```sh
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```sh
$ kubectl cluster-info
...
Kubernetes control plane is running at https://jin-vip:6443
CoreDNS is running at https://jin-vip:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

##### Join another controlplane 
Create token for control plane
```sh
## execute on jin-master-1

echo "$(kubeadm token create --print-join-command) --control-plane --certificate-key $(sudo kubeadm init phase upload-certs --upload-certs | grep -vw -e certificate -e Namespace)"

kubeadm join jin-vip:6443 --token <token> --discovery-token-ca-cert-hash <hash>  --control-plane --certificate-key <certificate_key>
```
Copy print command to **jin-master-2** and **jin-master-3**

##### Join worker to cluster
```sh
## execute on control plane (master 1/2/3)
sudo kubeadm token create --print-join-command

kubeadm join jin-vip:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

Print all node in cluster
```sh
$ kubectl get node
NAME           STATUS     ROLES           AGE     VERSION
jin-master-1   NotReady   control-plane   45m     v1.28.2
jin-master-2   NotReady   control-plane   2m54s   v1.28.2
jin-master-3   NotReady   control-plane   3m43s   v1.28.2
jin-worker-1   NotReady   <none>          16m     v1.28.2
```

##### Reconfiguring keepalived
change pgrep on **jin-master-1** from containerd to kube-apiserver
```sh
sudo sed -i 's/containerd/kube-apiserver/g' /etc/keepalived/keepalived.conf
```
Then we configure keepalived on jin-master-2 and jin-master-3 to get process id from `kube-apiserver`

<table>
<tr>
<th>Configuration Keepalived 2</th>
<th>Configuration Keepalived 3</th>
</tr>
<tr>
<td>
  
```sh
# sudo vi /etc/keepalived/keepalived.conf

### start from here
vrrp_script chk_api {
  script "/usr/bin/pgrep kube-apiserver"
  interval 1
  timeout 1
}

vrrp_instance kube-api {
  state BACKUP
  interface ens3 ## public interface
  advert_int 1
  priority 102
  virtual_router_id 51
  authentication {
      auth_type PASS
      auth_pass P4$$w0rd
  }
  unicast_src_ip 10.20.31.12
  unicast_peer {
    10.20.31.11
    10.20.31.13
  }
  virtual_ipaddress {
      10.20.31.10/24
  }
  track_script {
      chk_api
  }
}
```
  
</td>
<td>

```sh
# sudo vi /etc/keepalived/keepalived.conf

### start from here
vrrp_script chk_api {
  script "/usr/bin/pgrep kube-apiserver"
  interval 1
  timeout 1
}

vrrp_instance kube-api {
  state BACKUP
  interface ens3 ## public interface
  advert_int 1
  priority 101
  virtual_router_id 51
  authentication {
      auth_type PASS
      auth_pass P4$$w0rd
  }
  unicast_src_ip 10.20.31.13
  unicast_peer {
    10.20.31.11
    10.20.31.12
  }
  virtual_ipaddress {
      10.20.31.10/24
  }
  track_script {
      chk_api
  }
}
```

</td>
</tr>
</table>

##### Helm Installation
Preparing to install helm binary on **all master node**
```sh
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

##### Installing Cilium CNI
```sh
vi cilium-config.yaml
```
```yml
cluster:
  name: kubernetes
hubble:
  relay:
    enabled: true
  ui:
    enabled: true
k8sServiceHost: jin-vip
k8sServicePort: 6443
kubeProxyReplacement: strict
operator:
  replicas: 1
serviceAccounts:
  cilium:
    name: cilium
  operator:
    name: cilium-operator
tunnel: vxlan
```

```sh
helm repo add cilium https://helm.cilium.io/
helm install -n kube-system cilium cilium/cilium -f cilium-config.yaml
```

#### Verifying
```sh
$ kubectl get pod

NAMESPACE     NAME                                   READY   STATUS    RESTARTS      AGE
kube-system   cilium-gcj7h                           1/1     Running   0             8m8s
kube-system   cilium-gkvn9                           1/1     Running   0             8m8s
kube-system   cilium-k7j97                           1/1     Running   0             8m8s
kube-system   cilium-operator-5497f47bb9-jv56h       1/1     Running   0             8m8s
kube-system   cilium-vhs4r                           1/1     Running   0             8m8s
kube-system   coredns-5dd5756b68-4vnmc               1/1     Running   0             17h
kube-system   coredns-5dd5756b68-f9wtx               1/1     Running   0             17h
```

#### Reference
- [cilium for kubeadm (kubesimplify.com)](https://blog.kubesimplify.com/how-to-install-a-kubernetes-cluster-with-kubeadm-containerd-and-cilium-a-hands-on-guide)
