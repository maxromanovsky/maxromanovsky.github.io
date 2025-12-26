---
layout: post
title: "Running TP-Link Tapo cameras without access to internet"
category: local-mode
author: Maksym Romanowski
tags: [tp-link-tapo, camera, tl;dr]
---
## TL;DR
This snippet shows how to run TP-Link Tapo cameras fully locally, without access to internet.

## Environment
- TP-Link Tapo C110 hw 2.0 fw 1.4.7
- TP-Link Tapo C225 hw 2.0 fw 1.1.1
- NTP server running locally that doesn't use `time.windows.com` as a source of time

## Rationale
By default, TP-Link Tapo cameras are connected to cloud for multiple purposes:
- share video streams
- perform firmware updates
- sync time

Unfortunately, these cameras ignore NTP servers specified via DHCP config,
and seem to rely on a list of hardcoded NTP servers.
One of them is `time.windows.com`. You can find others in your firewall log once internet connectivity is blocked.

## Configuration
- Setup cameras via Tapo app with internet connected
    - Update firmware if necessary
    - Create RTSP account (otherwise what's the point of running it fully locally?)
    - Use DHCP or specify local DNS
- Block internet connection from cameras via network firewall (most likely on router)
- Create DNS `A` record (on DNS used by Tapo) pointing `time.windows.com` to local NTP server
