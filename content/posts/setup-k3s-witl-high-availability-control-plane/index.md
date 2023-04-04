---
title: "[K3s] Setup K3s with High Availability Control Plane"
date: 2023-04-03
draft: true
---

|    Hostname  |     IP         | Specification |
| ------------ | -------------  | ------------- |
| k3s-master-1 | 192.168.122.11 |   3Gb/2vcpu   |
| k3s-master-2 | 192.168.122.12 |   3Gb/2vcpu   |
| k3s-master-3 | 192.168.122.13 |   3Gb/2vcpu   |
| k3s-worker-1 | 192.168.122.21 |   8Gb/4vcpu   |
| k3s-worker-2 | 192.168.122.22 |   8Gb/4vcpu   |
| k3s-worker-3 | 192.168.122.23 |   8Gb/4vcpu   |


### Install package yang dibutuhkan (master-1)

#### kubectl
```sh
### download kubectl
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

### checksum
$ curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
$ echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

### install
$ sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

#### k3sup
```sh
### download
$ curl -sLS https://get.k3sup.dev | sh

### install
$ sudo install k3sup /usr/local/bin/
```

### Nodes

Dibutuhkan:

-   Static IP address
-   SSH key yang sudah di copy
-   Sudo setup untuk no password `%sudo ALL=(ALL:ALL) NOPASSWD: ALL`

## Deploy First Master Node

### Requirements & Information needed:

-   IP Address of node: `--ip = 192.168.122.11` (k3s-master-1)
-   IP Address to be used by kube-vip: `--tls-san = 192.168.122.100`
-   User with sudo priviledges: `--user ajinha`

### Deploy k3s:

_*Note: Pastikan  terdapat folder `$HOME/.kube/` **

```sh
### master 1
k3sup install \
--ip 192.168.122.11 \
--tls-san 192.168.122.100 \
--cluster \
--k3s-channel latest \
--context default \
--k3s-extra-args "--write-kubeconfig-mode 644 --disable servicelb --disable traefik --disable local-storage --disable metrics-server --service-cidr=10.96.0.0/16 --cluster-dns=10.96.0.10 --node-taint node-role.kubernetes.io/master=true:NoSchedule" \
--local-path $HOME/.kube/config \
--user ajinha
```

```sh
$ export KUBECONFIG=/home/$USER/.kube/config
```

### kube-vip using arp

```sh
curl -s https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml
export VIP=192.168.122.100
export INTERFACE=ens2
crictl pull docker.io/plndr/kube-vip:v0.3.5
alias kube-vip="ctr run --rm --net-host docker.io/plndr/kube-vip:v0.3.5 vip /kube-vip"

kube-vip manifest daemonset \
                  --arp \
                  --interface $INTERFACE \
                  --address $VIP \
                  --controlplane \
                  --leaderElection \
                  --taint \
                  --inCluster | tee /var/lib/rancher/k3s/server/manifests/kube-vip.yaml

ping $VIP
```


### kube-vip using bgp

```yaml
### sudo vi /var/lib/rancher/k3s/server/manifests/kube-vip.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  creationTimestamp: null
  name: kube-vip-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: kube-vip-ds
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: kube-vip-ds
    spec:
      containers:
      - args:
        - manager
        env:
        - name: vip_arp
          value: "false"
        - name: vip_interface
          value: "lo"
        - name: port
          value: "6443"
        - name: vip_cidr
          value: "32"
        - name: cp_enable
          value: "true"
        - name: cp_namespace
          value: kube-system
        - name: bgp_enable
          value: "true"
        - name: vip_loglevel
          value: "5"
        - name: bgp_routerinterface
          value: "ens2" # SESUAIKAN DENGAN INTERFACE PUBLIC
        - name: bgp_as
          value: "64500"
        - name: bgp_peeraddress
          value: "192.168.122.1" # SESUAIKAN DENGAN IP GATEWAY
        - name: bgp_peeras
          value: "64501"
        - name: vip_address
          value: "192.168.122.100" # SESUAIKAN DENGAN TLS-SAN SEBELUMNYA
        image: plndr/kube-vip
        imagePullPolicy: Always
        name: kube-vip
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - SYS_TIME
      hostNetwork: true
      nodeSelector:
        node-role.kubernetes.io/master: "true"
      serviceAccountName: kube-vip
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
        operator: Exists
  updateStrategy: {}
status:
  currentNumberScheduled: 0
  desiredNumberScheduled: 0
  numberMisscheduled: 0
  numberReady: 0
```

Test ping menggunakan virtual ip:
```sh
$ ping 192.168.122.100
```

Join master yang lainnya:
```sh
### master 2
k3sup join \
--ip 192.168.122.12 \
--server-ip 192.168.122.100 \
--server \
--k3s-channel latest \
--k3s-extra-args "--write-kubeconfig-mode 644 --disable servicelb --disable traefik --disable local-storage --disable metrics-server --service-cidr=10.96.0.0/16 --cluster-dns=10.96.0.10 --node-taint node-role.kubernetes.io/master=true:NoSchedule" \
--user ajinha

### master 3
k3sup join \
--ip 192.168.122.13 \
--server-ip 192.168.122.100 \
--server \
--k3s-channel latest \
--k3s-extra-args "--write-kubeconfig-mode 644 --disable servicelb --disable traefik --disable local-storage --disable metrics-server --service-cidr=10.96.0.0/16 --cluster-dns=10.96.0.10 --node-taint node-role.kubernetes.io/master=true:NoSchedule" \
--user ajinha
```

Join worker:
```sh
### worker 1
k3sup join \
--ip 192.168.122.21 \
--server-ip 192.168.122.100 \
--k3s-channel latest \
--user ajinha

### worker 2
k3sup join \
--ip 192.168.122.22 \
--server-ip 192.168.122.100 \
--k3s-channel latest \
--user ajinha

### worker 3
k3sup join \
--ip 192.168.122.23 \
--server-ip 192.168.122.100 \
--k3s-channel latest \
--user ajinha
```

### Add auto complete
```sh
### vi ~/.bashrc
...

source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k

...
```


## Source
- https://gist.github.com/rosswf/e5c4c85efb54a9f8d6e19a21cf09aa63
- https://github.com/ebrianne/k3s-at-home/blob/master/docs/theeasyway.md
- https://www.virtualizationhowto.com/2022/11/kube-vip-configuration-for-k3s-control-plane-ha/