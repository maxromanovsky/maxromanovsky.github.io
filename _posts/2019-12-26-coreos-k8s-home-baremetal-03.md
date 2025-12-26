---
layout: post
title: "Home pet cluster. Kubernetes on CoreOS. Part 3: Ingress"
category: kubernetes
author: Maksym Romanowski
tags: [coreos, kubernetes, kubespray, nuc, udoo, metallb, nginx, cert-manager, openwrt, haproxy, oauth2-proxy, helm]
outdated: true

---
My Kubernetes is up and running, and I've decided to expose certain services to the Internet, while keeping other services inside the home network.

<!--more-->

## 10000ft Overview

From the high level perspective it looks like this:
- External DNS configuration to point to the ISP-owned IP of my router (either using static IP or via DynDNS).
- Router configuration, one of the following:
    - OpenWRT router has HAProxy 2, which is configured to route clients supporting SNI either to K8s or to other targets (this is my case).
    - Simple port forwarding to route TCP traffic on port `443` to proper destination.
- K8s `LoadBalancer` (MetalLB) that would propagate custom IP address to the router. This allows scheduling LB on different nodes and failover.
- External and Internal Ingress Controller (Nginx) that would take care of k8s `Ingress` objects and create `LoadBalancer` services via MetalLB.
- OAuth2 Proxy configured in Nginx Ingress as external auth (for externally exposed services)
- K8s CoreDNS pointing to router DNS instead of Google DNS to allow inter-service communication via LAN (useful for Nginx <-> OAuth2 Proxy, for example)
- Cert Manager requesting SSL certificates that are used by Ingress Controller

User request (assuming HTTPS traffic to K8s service using client supporting SNI) originating from the Internet is routed via:
- External DNS pointing to the router IP
- HAProxy 2
- LAN IP address associated with external MetalLB `LoadBalancer` associated with MAC address of one of the nodes network card
- `kube-proxy` spreading traffic between one of `metallb-speaker`s
- Nginx with proper SSL certificates performs SSL termination
- Nginx performs authentication via OAuth2 Proxy (K8s CoreDNS uses OpenWRT DNS to lookup the IP address, and then request is forwarded (again) via MetalLB and nginx)
- Service and Pod networking do their magic
- HTTP traffic finally reaches the pod

![10000ft Overview](/assets/images/k8s-coreos-home-baremetal/k8s-ingress.png)

# OpenWRT

## HAProxy
OpenWRT router is configured with [HAProxy 2](http://www.haproxy.org/) listening on port `443` with SNI configuration allowing to proxy certain domain names to Kubernetes, while other to another target.

If Kubernetes (on a single IP) is the only destination, simple port forwarding can be used instead.

Here's snippet of HAProxy config, including web UI and prometheus metrics:

{% raw %}
```
global
    daemon

defaults
    timeout client 30s
    timeout server 30s
    timeout connect 5s

frontend ft_ssl
  bind :443
  mode tcp
  tcp-request inspect-delay 5s
  tcp-request content accept if { req_ssl_hello_type 1 }

  acl k8s_app_1 req_ssl_sni -i app1.k8s.example.com
  acl k8s_app_2 req_ssl_sni -i app2.k8s.example.com

  use_backend bk_ssl_k8s_x64 if k8s_app_1
  use_backend bk_ssl_k8s_x64 if k8s_app_2

  default_backend bk_ssl_non_k8s

  # K8s x64
backend bk_ssl_k8s_x64
  mode tcp
  balance roundrobin
  stick-table type binary len 32 size 30k expire 30m
  acl clienthello req_ssl_hello_type 1
  acl serverhello rep_ssl_hello_type 2
  tcp-request inspect-delay 5s
  tcp-request content accept if clienthello
  tcp-response content accept if serverhello
  stick on payload_lv(43,1) if clienthello
  stick store-response payload_lv(43,1) if serverhello
  option ssl-hello-chk
  server k8s-x64-node1 192.168.0.3:443 check
  #server k8s-x64-node2 192.168.0.4:443 check

backend bk_ssl_non_k8s
  mode tcp
  balance roundrobin
  stick-table type binary len 32 size 30k expire 30m
  acl clienthello req_ssl_hello_type 1
  acl serverhello rep_ssl_hello_type 2
  tcp-request inspect-delay 5s
  tcp-request content accept if clienthello
  tcp-response content accept if serverhello
  stick on payload_lv(43,1) if clienthello
  stick store-response payload_lv(43,1) if serverhello
  option ssl-hello-chk
  server non_k8s 192.168.0.2:443

listen stats
    bind *:1936
    http-request use-service prometheus-exporter if { path /metrics }
    mode http
    stats enable
    stats uri /
    stats refresh 10s
```
{% endraw %}

## Internal DNS

Router's DNS is used to avoid routing traffic with both source and destination within the LAN.
It maps hostnames to the IP addresses of Kubernetes ingress services (`type: LoadBalancer`).

{% raw %}
```bash
uci add dhcp domain
uci set dhcp.@domain[-1].name='app1.k8s.example.com'
uci set dhcp.@domain[-1].ip='192.168.0.3'

uci add dhcp domain
uci set dhcp.@domain[-1].name='app2.k8s.example.com'
uci set dhcp.@domain[-1].ip='192.168.0.3'

uci commit
luci-reload
service dnsmasq restart
```
{% endraw %}

## Kubernetes

I was not a huge fan of Helm 2, especially due to the Tiller component. But also due to many charts using outdated `apiVersion` in their templates, but now as K8s API became much more mature, I am enjoying Helm 3 instead. That's why most of the services in this post are deployed via Helm.

### Kubespray config


Several Kubespray configuration options have to be adjusted to make this setup working.

One is related to configure Kubernetes DNS to use OpenWRT DNS as an upstream instead of Google DNS.

Another one is about enabling scheduling on a master node of my two-node cluster.

Also, I used this as a chance to convert inventory to YAML, which allows me to configure node labels :)

Additionally, I've configured several components to expose their metrics in Prometheus format.

Inventory:

{% raw %}
```yaml
all:
  hosts:
    nuc:
      ansible_host: 192.168.0.10
      ansible_user: core
      etcd_member_name: nuc5ppyh
      node_labels: {'disktype': 'hdd', 'cputype': 'pentium'}

    udoo:
      ansible_host: 192.168.0.11
      ansible_user: core
      node_labels: {'disktype': 'ssd', 'cputype': 'celeron'}

  children:
    etcd: {hosts: {nuc: {}}}
    k8s-cluster:
      children:
        kube-master: {hosts: {nuc: {}}}
        # Master node must be included into kube-node to avoid NoSchedule taint
        kube-node: {hosts: {nuc: {}, udoo: {}}}
```
{% endraw %}

Configuration:
{% raw %}
```yaml
## Upstream dns servers
upstream_dns_servers:
 - 192.168.0.1
# - 8.8.8.8
# - 8.8.4.4

## Prometheus metrics
# The IP address and port for the metrics server to serve on
# (set to 0.0.0.0 for all IPv4 interfaces and `::` for all IPv6 interfaces)
kube_proxy_metrics_bind_address: 0.0.0.0:10249

calico_felix_prometheusmetricsenabled: true
calico_felix_prometheusmetricsport: 9091
calico_felix_prometheusgometricsenabled: true
calico_felix_prometheusprocessmetricsenabled: true
```
{% endraw %}

Here's quick way to verify that changes were correctly applied:

{% raw %}
```bash
# Checking labels
kubectl get node --show-labels

# Checking that NoSchedule was removed from master
kubectl get nodes -o json | jq '.items[].spec.taints'

# Checking that K8s DNS uses local DNS as an upstream
kubectl -nkube-system get cm coredns -oyaml | grep upstream
kubectl -nkube-system get cm nodelocaldns -oyaml | grep forward

# Checking that Prometheus metrics are accessible
kubectl -n kube-system get cm kube-proxy -oyaml | grep metricsBindAddress
curl 192.168.0.10:9091/metrics
curl 192.168.0.11:9091/metrics
```
{% endraw %}

### MetalLB

[MetalLB](https://metallb.universe.tf/) is an awesome little load balancer implementation for both enthusiasts (like myself) and serious production deployments running on bare metal.

I neither have fancy hardware to support BGP, nor actually need it.

[Layer 2](https://metallb.universe.tf/concepts/layer2/) (ARP) mode and proxying all the traffic using single node (with an automatic failover, thanks to ARP!) works for me!

Without further ado, here's my Helm 3 configuration for MetalLB. It matches IP addresses that I use for OpenWRT internal DNS, as well as for HAProxy configuration. That's the key :smile:

{% raw %}
```yaml
configInline:
  address-pools:
  # Used in HAProxy and internal DNS, for services accessible from Internet
  - name: external
    protocol: layer2
    addresses:
    - 192.168.0.3/32

  # Used only in internal DNS, for services accessible from LAN only
  - name: internal
    protocol: layer2
    addresses:
    - 192.168.0.5/32

# Of course, Prometheus!
prometheus:
  serviceMonitor:
    enabled: true
  prometheusRule:
    enabled: true
```
{% endraw %}

### Nginx Ingress Controller

Nginx Ingress Controller have to be configured to create proper `LoadBalancer` service, and `Ingress` objects must map to proper ingress controller and include Authn configuration.

Here's Helm configuration for external Nginx Ingress (keep an eye on [MetalLB annotation](https://metallb.universe.tf/usage/#requesting-specific-ips)!):
{% raw %}
```yaml
controller:
  name: controller-external
  electionID: ingress-controller-external-leader
  ingressClass: nginx-external
  kind: DaemonSet
  service:
    omitClusterIP: true
    annotations:
      metallb.universe.tf/address-pool: external
  metrics:
    enabled: true
    service:
      omitClusterIP: true

defaultBackend:
  service:
    omitClusterIP: true
```
{% endraw %}

Here's sample `Ingress` object leveraging Nginx Ingress [external OAuth support](https://kubernetes.github.io/ingress-nginx/examples/auth/oauth-external-auth/) and SSL certificates managed via Cert Manager:

{% raw %}
```yaml
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: app1
  namespace: app1
  annotations:
    kubernetes.io/ingress.class: "nginx-external"
    nginx.ingress.kubernetes.io/auth-url: "https://$host/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://$host/oauth2/start?rd=$escaped_request_uri"
spec:
  tls:
    - hosts:
        - app1.k8s.example.com
      secretName: tls-app1
  rules:
    - host: app1.k8s.example.com
      http:
        paths:
          - backend:
              serviceName: app1
              servicePort: 80
            path: /

```
{% endraw %}

### OAuth2 Proxy

[OAuth2 Proxy](https://pusher.github.io/oauth2_proxy/) is another wonderful service transparently enabling OAuth2 for applications that don't support it natively. It works great with Nginx Ingress too.

Helm configuration:
{% raw %}
```yaml
config:
  clientID: "app1-cid"
  clientSecret: "app1-csec"
  # openssl rand -base64 32 | head -c 32
  cookieSecret: "cookieMonsterAteTheSecret"
  configFile: |-
    provider = "foooo"
    upstream = "file:///dev/null"
    footer = "-"

ingress:
  enabled: true
  path: /oauth2
  hosts:
  - app1.k8s.example.com
  annotations:
    kubernetes.io/ingress.class: "nginx-external"
  tls:
  - secretName: tls-app1
    hosts:
    - app1.k8s.example.com

```
{% endraw %}

# Conclusion

That's about it, folks. Now I can easily and (more or less) securely access my k8s services from both LAN and Internet using the same host names, proper SSL certificates and OAuth 2 support.
