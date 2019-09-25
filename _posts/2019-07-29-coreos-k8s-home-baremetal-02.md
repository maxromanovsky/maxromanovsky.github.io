---
layout: post
title: "Home pet cluster. Kubernetes on CoreOS. Part 2: Spraying some kubes with Kubespray"
category: kubernetes
author: Maksim Ramanouski
tags: [coreos, kubernetes, kubespray, nuc, udoo]

---
At this point I have two Linux machines running CoreOS Container Linux.

Now it's time to finally install Kubernetes on them!

<!--more-->

On a side note, Fedora released [the first preview](https://fedoramagazine.org/introducing-fedora-coreos/) of Fedora CoreOS, which is going to replace OSS [CoreOS Container Linux](https://unix.stackexchange.com/a/491237), a.k.a. CoreOS. However, I've decided to stick with CoreOS Container Linux, as I want to have at least one stable component in my system :smile:.

[Kubespray](https://github.com/kubernetes-sigs/kubespray/) is an official tool for deploying a Production-Ready Kubernetes cluster on wide variety of infrastructure, including bare metal. It is based on Ansible, and uses kubeadm under the hood (if I'm not mistaken).

I've used [v2.10.4](https://github.com/kubernetes-sigs/kubespray/releases/tag/v2.10.4), which was latest at the moment. Personally, I configured it as a submodule to my git repo with custom configs, but it could be just downloaded from a tag, and used independently of git.

[Quickstart](https://kubespray.io/#/?id=usage) usage commands are available on it's website, and I won't repeat them. However, I'd like to emphasize that you should use an Ansible from kubespray's `requirements.txt` instead of latest & greatest from your package manager. It turns out, that Ansible sometimes breaks backward compatibility or introduces bugs, which break kubespray. It happend to me twice (in 2018 and in 2019), and both times Ansible pinned to Kubespray worked like a charm.

[CoreOS](https://kubespray.io/#/docs/coreos) is supported, and main `cluster.yml` playbook even auto-detects CoreOS and sets proper defaults, however other playbooks (such as `reset.yml`) don't support them. Following changes worked for me, when I've configured, and then tore down the cluster multiple times.

## Configuration
{% raw %}
```diff
diff --git a/k8s/inventory-x64/group_vars/all/all.yml b/k8s/inventory-x64/group_vars/all/all.yml
index 4b45b66..d16c4c2 100644
--- a/k8s/inventory-x64/group_vars/all/all.yml
+++ b/k8s/inventory-x64/group_vars/all/all.yml
@@ -3,7 +3,9 @@
 etcd_data_dir: /var/lib/etcd

 ## Directory where the binaries will be installed
-bin_dir: /usr/local/bin
+bin_dir: /opt/bin
+
+ansible_python_interpreter: /opt/bin/python

 ## The access_ip variable is used to define how other nodes should access
 ## the node.  This is used in flannel to allow other flannel nodes to see
@@ -39,9 +41,9 @@ loadbalancer_apiserver_healthcheck_port: 8081
 # kubelet_load_modules: false

 ## Upstream dns servers
-# upstream_dns_servers:
-#   - 8.8.8.8
-#   - 8.8.4.4
+upstream_dns_servers:
+ - 8.8.8.8
+ - 8.8.4.4

 ## There are some changes specific to the cloud providers
 ## for instance we need to encapsulate packets with some network plugins
diff --git a/k8s/inventory-x64/group_vars/k8s-cluster/k8s-cluster.yml b/k8s/inventory-x64/group_vars/k8s-cluster/k8s-cluster.yml
index b2bfdf0..dcd9fcc 100644
--- a/k8s/inventory-x64/group_vars/k8s-cluster/k8s-cluster.yml
+++ b/k8s/inventory-x64/group_vars/k8s-cluster/k8s-cluster.yml
@@ -124,7 +124,7 @@ kube_encrypt_secret_data: false

 # DNS configuration.
 # Kubernetes cluster name, also will be used as DNS domain
-cluster_name: cluster.local
+cluster_name: your_cluster_name.local
 # Subdomains of DNS domain to be resolved via /etc/resolv.conf for hostnet pods
 ndots: 2
 # Can be coredns, coredns_dual, manual or none
@@ -136,7 +136,7 @@ enable_nodelocaldns: true
 nodelocaldns_ip: 169.254.25.10

 # Can be docker_dns, host_resolvconf or none
-resolvconf_mode: docker_dns
+resolvconf_mode: host_resolvconf
 # Deploy netchecker app to verify DNS resolve as an HTTP service
 deploy_netchecker: false
 # Ip address of the kubernetes skydns service
@@ -175,9 +175,9 @@ dynamic_kubelet_configuration_dir: "{{ kubelet_config_dir | default(default_kube
 podsecuritypolicy_enabled: false

 # Make a copy of kubeconfig on the host that runs Ansible in {{ inventory_dir }}/artifacts
-# kubeconfig_localhost: false
+kubeconfig_localhost: true
 # Download kubectl onto the host that runs Ansible in {{ bin_dir }}
 # kubectl_localhost: false
```
{% endraw %}

Couple notes about those changes:

- You need to specify `upstream_dns_servers`, otherwise DNS resolution won't work, and Kubespray will even fail to download containers to bootstrap the cluster. The issue [#2831](https://github.com/kubernetes-sigs/kubespray/issues/2831) was reported in kubespray about that (but closed unresolved due to inactivity).
- I prefer changing `cluster_name` to something meaningful. Pick your own name for a pet cluster :pig:


My inventory file is pretty straightforward. My cluster doesn't have HA for K8s master and etcd, as it has just 2 nodes.

{% raw %}
```ini
nuc ansible_host=192.168.0.10 ansible_user=core etcd_member_name=nuc5ppyh
udoo ansible_host=192.168.0.11 ansible_user=core

[kube-master]
nuc

[etcd]
nuc

[kube-node]
udoo

[k8s-cluster:children]
kube-master
kube-node

```
{% endraw %}

## Cluster provisioning

Provisioning requires just a single command, but prepare for a long process :clock1:

{% raw %}
```bash
ansible-playbook -i inventory-x64/inventory.ini kubespray-x64/cluster.yml -b -v > install.log
```
{% endraw %}

`-b` stands for `become`, which means `sudo` to CoreOS nodes in our case, while `-v` produces a verbose output to the `install.log` file. If cluster provisioning fails, look at the last lines of this log file.

Once provisioning is completed, copy `artifacts/admin.conf` to `~/.kube/config` file and install `kubectl` on your machine.

Now (hopefully) you have a running cluster, and you can verify it by invoking some kubectl commands:

{% raw %}
```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.1", GitCommit:"4485c6f18cee9a5d3c3b4e523bd27972b1b53892", GitTreeState:"clean", BuildDate:"2019-07-18T14:25:20Z", GoVersion:"go1.12.7", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.3", GitCommit:"5e53fd6bc17c0dec8434817e69b04a25d8ae0ff0", GitTreeState:"clean", BuildDate:"2019-06-06T01:36:19Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}

$ kubectl cluster-info
Kubernetes master is running at https://192.168.0.10:6443
coredns is running at https://192.168.0.10:6443/api/v1/namespaces/kube-system/services/coredns:dns/proxy
kubernetes-dashboard is running at https://192.168.0.10:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
nuc    Ready    master   28d   v1.14.3
udoo   Ready    <none>   28d   v1.14.3

$ kubectl get all --all-namespaces
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
kube-system   pod/calico-kube-controllers-79c7fd7f68-cshvb   1/1     Running   3          28d
kube-system   pod/calico-node-bzc7t                          1/1     Running   3          28d
kube-system   pod/calico-node-sr6x8                          1/1     Running   3          28d
kube-system   pod/coredns-56bc6b976d-2zslx                   1/1     Running   3          28d
kube-system   pod/coredns-56bc6b976d-p9jql                   1/1     Running   3          28d
kube-system   pod/dns-autoscaler-5fc5fdbf6-f4zq7             1/1     Running   3          28d
kube-system   pod/kube-apiserver-nuc                         1/1     Running   4          28d
kube-system   pod/kube-controller-manager-nuc                1/1     Running   4          28d
kube-system   pod/kube-proxy-6xnwn                           1/1     Running   3          28d
kube-system   pod/kube-proxy-pqfcw                           1/1     Running   3          28d
kube-system   pod/kube-scheduler-nuc                         1/1     Running   4          28d
kube-system   pod/kubernetes-dashboard-6c7466966c-lmh72      1/1     Running   4          28d
kube-system   pod/nginx-proxy-udoo                           1/1     Running   3          28d
kube-system   pod/nodelocaldns-7pfm2                         1/1     Running   3          28d
kube-system   pod/nodelocaldns-k8kll                         1/1     Running   3          28d


NAMESPACE     NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes             ClusterIP   10.233.0.1     <none>        443/TCP                  28d
kube-system   service/coredns                ClusterIP   10.233.0.3     <none>        53/UDP,53/TCP,9153/TCP   28d
kube-system   service/kubernetes-dashboard   ClusterIP   10.233.5.252   <none>        443/TCP                  28d

NAMESPACE     NAME                          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
kube-system   daemonset.apps/calico-node    2         2         2       2            2           <none>                        28d
kube-system   daemonset.apps/kube-proxy     2         2         2       2            2           beta.kubernetes.io/os=linux   28d
kube-system   daemonset.apps/nodelocaldns   2         2         2       2            2           <none>                        28d

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/calico-kube-controllers   1/1     1            1           28d
kube-system   deployment.apps/coredns                   2/2     2            2           28d
kube-system   deployment.apps/dns-autoscaler            1/1     1            1           28d
kube-system   deployment.apps/kubernetes-dashboard      1/1     1            1           28d

NAMESPACE     NAME                                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/calico-kube-controllers-79c7fd7f68   1         1         1       28d
kube-system   replicaset.apps/coredns-56bc6b976d                   2         2         2       28d
kube-system   replicaset.apps/dns-autoscaler-5fc5fdbf6             1         1         1       28d
kube-system   replicaset.apps/kubernetes-dashboard-6c7466966c      1         1         1       28d
```
{% endraw %}

Long story short, it's Kubernetes v1.14.3 with the following components:
- Calico for pod networking
- CoreDNS as Kubernetes DNS server
- Nginx as proxy to API Server for worker nodes (makes it available on `localhost:6443`), and is called a [localhost loadbalancing](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ha-mode.md#kube-apiserver) in Kubespray docs.
- Kubernetes web-based dashboard

One important thing we're missing is an Ingress for an external acces to cluster, but we'll configure it next time.
