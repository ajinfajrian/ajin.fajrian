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
#draft: true
---

Hello. Just finished my playground lab a few days ago, and it's been long time not write some posts to internet. 
The reason i build this cluster with kube-vip bgp mode is this tools can provides high availability and load-balancing and also at the same time i want to learn about bgp routing can advertising virtual ip.


## Topology

```
                                                                                           
                                                                                           
                                   +--------------------+                                  
                                   |   jin-bgp-router   |                                  
                     +-------------|    10.20.31.51     |--------------+                   
                     |             +--------------------+              |                   
                     |                        |     bgp routes         |                   
          asn: 65000 |             asn: 65000 |    advertisement:      |asn: 65000         
                     |                        |    30.30.30.0/24       |                   
                     |                        |                        |                   
          +--------------------+   +--------------------+   +--------------------+         
          |  jin-k8s-master-1  |   |  jin-k8s-master-2  |   |  jin-k8s-master-3  |         
          |  ens3:10.20.31.11  |   |  ens3:10.20.31.12  |   |  ens3:10.20.31.11  |         
          |  lo  :30.30.30.10  |   |  lo  :30.30.30.10  |   |  lo  :30.30.30.10  |         
          +--------------------+   +--------------------+   +--------------------+         
                     |                        |                        |                   
                     |                        |                        |                   
                     |             +--------------------+              |                   
                     |             |    kube-vip bgp    |              |                   
                     +-------------|    30.30.30.10     |--------------+                   
                                   +--------------------+                                  
                                              |                                            
                                              |                                            
                                              |                                            
                                   +--------------------+                                  
                                   |  jin-k8s-worker-1  |                                  
                                   |  ens3:10.20.31.20  |                                  
                                   +--------------------+                                  
```

## Prerequisites
- 1 bird bgp instance
- 3 control plane nodes
- 1 worker nodes


## Setup BGP

## Setup Depedencies

## Bootstraping Kubernetes


## Conclusion

## References
- [Say good-bye to HAProxy and Keepalived with kube-vip on your HA K8s control plane](https://inductor.medium.com/say-good-bye-to-haproxy-and-keepalived-with-kube-vip-on-your-ha-k8s-control-plane-bb7237eca9fc)