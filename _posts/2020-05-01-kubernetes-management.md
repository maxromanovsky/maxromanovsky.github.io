---
layout: post
title: "My approach to Kubernetes installation & management on bare metal"
category: kubernetes
author: Maksym Romanowski
tags: [kubernetes, flatcar, coreos, hypriotos, kubespray, k3s, pulumi, helmsman, helm]
outdated: true
---

## Installation

- `x86_64`:
    - CoreOS (now [Flatcar Container Linux](https://www.flatcar-linux.org/)) [as a Linux distro]({% post_url 2019-07-06-coreos-k8s-home-baremetal-01 %})
    - [Kubespray](https://kubespray.io/) as a [Kubernetes installer]({% post_url 2019-07-29-coreos-k8s-home-baremetal-02 %})
    - [Metallb](https://metallb.universe.tf/) and [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) for [incoming traffic]({% post_url 2019-12-26-coreos-k8s-home-baremetal-03 %})
- `arm` (Raspberry PI 3B)
    - [HypriotOS](https://github.com/hypriot/image-builder-rpi/releases) as a lightweight container-oriented Debian-based Linux
    - [k3s](https://k3s.io/) as a lightweight Kubernetes distribution with `sqlite` instead of `etcd`
    - [Metallb](https://metallb.universe.tf/) and [Traefik v1](https://containo.us/traefik/) for incoming traffic

## Configuration

- [Pulumi](https://www.pulumi.com/) for [everything except Helm charts]({% post_url 2020-04-21-pulumi %})
- [Helmsman](https://github.com/Praqma/helmsman) for [Helm charts]({% post_url 2020-04-24-helmsman %})
- [kubie](https://github.com/sbstp/kubie) for using multiple Kubernetes contexts simultaneously in different terminals
