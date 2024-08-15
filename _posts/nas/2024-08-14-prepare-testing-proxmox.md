---
title: Proxmox VE 8.2.4
author: tanezky
date: 2024-08-14 20:00:00 +/-TTTT
description: Prepare to test Proxmox Virtual Environment
categories: [NAS]
tags: [NAS,Terramaster,F4-424 Pro, Proxmox]
image: /assets/img/posts/pve-preparation/pve-prep-post.jpg
---

## Proxmox Virtual Environment (PVE)
![Proxmox PVE](/assets/img/posts/pve-preparation/pve-prep01.jpg){: width="1800" height="1070" .center}
_PVE Summary Dashboard_
After reading about Proxmox I got excited. It's a virtualization platform built on Debian, though it utilizes a modified Ubuntu LTS kernel. Proxmox uses LXC (Linux containers) for containerization and KVM (kernel-based virtual machine) for running virtual machines. It also has a full software-defined networking (SDN) stack and is compatible with secure boot.

I'll be conducting various experiments to familiarize myself with PVE's workings. There seem to be multiple ways to achieve the same outcomes, so I'll document the basic installation process in this article and future posts will delve into the results of my experiments.

For this exploration, I'll be using the latest stable version Proxmox VE 8.2.4, which is based on Debian Bookworm.


## Installing the OS
![Proxmox PVE](/assets/img/posts/pve-preparation/pve-prep02.jpg){: width="1800" height="1070" .center}
For the initial installation I'll install PVE on my Samsung NVMe drive and use the ext4 filesystem. I'm more accustomed to ext4 but after some experimentation I plan to try installing PVE on ZFS.


## Configuring Update Channels
![Proxmox Update Channels](/assets/img/posts/pve-preparation/pve-prep03.jpg){: width="1800" height="1070" .center}
_Configuration from the web-ui_
Since I'm using the community edition I'm not eligible for enterprise updates. To ensure I receive the appropriate updates I need to configure the system to use the `pve-no-subscription` repository and disable the enterprise repositories. This can be done in two ways: through the web UI or by directly editing the APT source lists.

From web-ui navigate to the Repositories page, disable enterprise channels and add `No-Subscription` repository.
Hit Reload and go to `Updates` page to `Refresh` and `Upgrade`. 

Here's the command I used to disable the enterprise repositories, add the pve-no-subscription repo to a separate file and update the package lists:
```shell
sed -i 's/^deb/# deb/' /etc/apt/sources.list.d/pve-enterprise.list && \
sed -i 's/^deb/# deb/' /etc/apt/sources.list.d/ceph.list && \
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list && \
apt update
```
![Proxmox List](/assets/img/posts/pve-preparation/pve-prep04.jpg){: width="1800" height="1070" .center}
_pve-no-subscription shows up in separate file_
I prefer using `sources.list` exclusively for Debian-related updates and adding other sources to their own dedicated files.