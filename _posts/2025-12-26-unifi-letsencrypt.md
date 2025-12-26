---
layout: post
title: "Adding Let's Encrypt TLS certificate to UniFi"
category: unifi
author: Maksym Romanowski
tags: [unifi, letsencrypt, tl;dr]
---
## TL;DR
This snippet installs Let's Encrypt TLS certificate, ensures it is renewed, and makes sure that UniFi is resolved to LAN IP when accessed via FQDN in TLS certificate.

## Environment
- UniFi OS 4.4.6
- UniFi Network 10.0.162
- [kchristensen/udm-le commit `316a8509063ce5161436c49a844da18de2681d27`](https://github.com/kchristensen/udm-le/commit/316a8509063ce5161436c49a844da18de2681d27)

## Configuration
- Follow installation instructions in [udm-le Readme](kchristensen/udm-le/commit/316a8509063ce5161436c49a844da18de2681d27)
- Add `A` DNS record pointing to UniFi device LAN IP in Policy Engine
