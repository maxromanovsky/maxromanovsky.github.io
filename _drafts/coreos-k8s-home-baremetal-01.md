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

I have two x64 mini computers, which are good candidates for Kubernetes nodes. They are not equal, but powerful enough. And they are unique to me, not only because they're so different :cat:

That's why I still treat them as pets, not cattle.

![Pets vs cattle](/assets/images/k8s-coreos-home-baremetal/pet-cattle.jpg) 

Image from [StackExchange](https://devops.stackexchange.com/questions/653/what-is-the-definition-of-cattle-not-pets)

One of them is called Nuc, and it's breed is Intel [NUC5PPYH](https://www.intel.com/content/www/us/en/products/boards-kits/nuc/kits/nuc5ppyh.html).

![NUC5PPYH](/assets/images/k8s-coreos-home-baremetal/nuc5ppyh.jpg)

Image from [Amazon](https://www.amazon.com/Intel-Nuc5ppyh-Components-Silver-BOXNUC5PPYH/dp/B00XPVQHDU)

According to [spec](https://ark.intel.com/content/www/us/en/ark/products/87740/intel-nuc-kit-nuc5ppyh.html), it has 4-core Pentium [N3700](https://ark.intel.com/content/www/us/en/ark/products/87261/intel-pentium-processor-n3700-2m-cache-up-to-2-40-ghz.html) and supports up to 8Gb RAM. Mine has 8Gb RAM and 1Tb HDD.

Another one responds to Udoo, and it's breed is Udoo x86 (first version from [Kickstarter](https://www.kickstarter.com/projects/udoo/udoo-x86-the-most-powerful-maker-board-ever)).

![Udoo x86](/assets/images/k8s-coreos-home-baremetal/udoo.png)

Image from [Kickstarter](https://www.kickstarter.com/projects/udoo/udoo-x86-the-most-powerful-maker-board-ever)

Mine version is called UDOO X86 Advanced, and it has 4-core Celeron [N3160](https://ark.intel.com/content/www/us/en/ark/products/91831/intel-celeron-processor-n3160-2m-cache-up-to-2-24-ghz.html), as well as 4Gb of RAM, but rather small 8Gb eMMC.

Both processors are [quite similar](https://ark.intel.com/content/www/us/en/ark/compare.html?productIds=91831,87261) in their performance, but Nuc has twice more memory, and large (but slower) HDD.

# CoreOS installation

Both pets have Ethernet port, and I prefer stable wired connection with DHCP address reservation configured to their MAC addresses. But here's the thing, it's either Ethernet or monitor in my apartment :smile:. I had to choose either to plug the physical screen or the network. This somewhat drove the approach I've used for OS installation.

I've used two USB drives:
- One with the [latest stable CoreOS ISO](https://coreos.com/os/docs/latest/booting-with-iso.html) flashed by [Balena Etcher](https://www.balena.io/etcher/). This ISO would be used to boot the CoreOS and run `coreos-install`.
- Another with couple of goodies:
    - [Latest stable](https://stable.release.core-os.net/amd64-usr/current/coreos_production_image.bin.bz2) `coreos_production_image.bin.bz2` (not to be confused with the ISO). That would be actually an image that would be installed to the machine.
    - `ignition.json`, a CoreOS config file consumable by `coreos-install`.
    -  binary, that would turn `yaml` config to less-readable `json` used by `coreos-install` to configure OS installation.

Here's the process I've followed to install the OS:
- Generate a hash from the password:
    - `mkpasswd --method=SHA-512 --rounds=4096` should be used, as shown in [an example](https://coreos.com/os/docs/latest/clc-examples.html#generating-a-password-hash)
    - I'm using the password for authentication as an alternative to SSH pubkey to log into the machine interactively, while it's still connected to the screen, not network.
- Write `ignition.yaml`, a human-readable version of CoreOS config file that contains:
    - Password hash
    - SSH pubkeys
    - CoreOS autoupdate strategy
- I store my `ignition.yaml` in the repo (without password hash and pubkey) as a reference, but CoreOS installer uses another config format, which is a less-readable `json`. Latest [Config Transpiler](https://github.com/coreos/container-linux-config-transpiler/releases) should be downloaded to create the JSON config: `ct < ignition.yaml > ignition.json`. There's even a [validation service](https://coreos.com/validate/) to check that config is well-formed. 
- Resulting config is written to a USB drive (different from the one where bootable ISO is flashed), alongside with the `bin.bz2` image.

