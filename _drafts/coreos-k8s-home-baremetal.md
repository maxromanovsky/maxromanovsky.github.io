---
layout: post
title: "Home pet cluster. Kubernetes on CoreOS. Part 1: don't call us cattle!"
category: kubernetes
author: Maksim Ramanouski
tags: [coreos, kubernetes, kubespray, nuc, udoo]

---
I always wanted to run a small Kubernetes cluster at home.

Why not in cloud? Kubernetes in cloud is still expensive if it's just for fun. And I have some mini computer at home, as well as desire to look how kubernetes works on the network layer.

I was never satisfied with "fat" Linux distributions for running containers, and didn't want to configure `unattended-upgrades`. Trying out CoreOS Container Linux sounded like a natural fit for my little pet.

<!--more-->

Anyway, I've decided to start with this:
- Hardware
    - Intel [NUC5PPYH](https://www.intel.com/content/www/us/en/products/boards-kits/nuc/kits/nuc5ppyh.html)
    - UDOO [x86](https://www.udoo.org/udoo-x86/) (I have the first version)
- Software
    - CoreOS [Container Linux](https://coreos.com/os/docs/latest/)
    - [Kubespray](https://kubespray.io/) v2.10.4

# Hardware 

issue with resolv.conf on [v2.10.4](https://github.com/kubernetes-sigs/kubespray/issues/2831)




![NUC5PPYH](/assets/images/k8s-coreos-home-baremetal/nuc5ppyh.jpg)
https://www.amazon.com/Intel-Nuc5ppyh-Components-Silver-BOXNUC5PPYH/dp/B00XPVQHDU

```json
{"foo":  "bar"}
```

