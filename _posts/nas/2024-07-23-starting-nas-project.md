---
title: Planning - NAS Project & Requirements
author: tanezky
date: 2024-07-23 17:00:00 +/-TTTT
description: Beginning of NAS Project
categories: [NAS]
tags: [NAS]
image: /assets/img/posts/project_post.jpg
---

## Motivation

My trusty old i5 desktop PC has served as my homelab server faithfully for over a decade. However, it's time for an upgrade. I'm looking for something more energy-efficient, with a smaller footprint that better suits my apartment space. Additionally, the current storage is insufficient for my family's growing needs for file syncing and disaster recovery.

## Requirements

I had a few key requirements in mind for the NAS:

- **Processor** A multi-core processor with virtualization support to handle multiple tasks and potentially run virtual machines.
- **Memory** At least 16 GB of RAM to ensure smooth operation of services and leave some room for additional experimentation in my homelab.
- **Capacity** A minimum of 16 TB of usable disk space to accommodate current and future storage needs.
- **Redundancy** Disk failure protection (software RAID-Z)
- **Power Consumption** Good power efficiency to reduce energy usage and lower my electricity bill.
- **Security** Support for FTPM (Firmware Trusted Platform Module) to ensure secure boot and protect against unauthorized firmware modifications.
- **Encryption** All data is encrypted.
- **Size** A compact form factor, smaller than ATX.
- **Ethernet** At least 2.5 Gbs

## CPU Comparison

|               | Old           | [Terramaster F4-424 Pro](https://www.terra-master.com/global/f4-424-pro.html)    | [Synology DiskStation DS923+](https://www.synology.com/en-us/products/DS923+) |
| ---           | ---           | ---                       | --- |
| **CPU**       | Intel i5-661  | Intel i3-N305             | AMD Ryzen R1600 |
| **Generation**| 1st           | 12th                      | Zen |
| **Max Freq**  | 3.6 GHz       | 3.8 GHz                   | 3.1 GHz |
| **Cores (threads)**| 2 (4)      | 8E (8)[^fn1]            | 4 (8)            
| **TDP**       | 87 W          | 15 W                      | 25 W |
| [**Passmark Score**](https://www.cpubenchmark.net/compare/769vs5213vs5117/Intel-i5-661-vs-Intel-i3-N305-vs-AMD-Ryzen-Embedded-R1600) | 2469     | 10055                     | 3365 |
| **Installed Memory**    | 16 GB         | 32 GB                     | 4 GB (max 32 GB 2x16 GB) |
| **Ethernet**    | 1 x 1GBASE-T <br>(upgradable via X1 PCIe)  | 2 x 2.5GBASE-T      | 2 x 1GBASE-T<br>(upgradable via proprietary X2 PCIe) | 

I'd been considering the Synology DiskStation DS923+, but the Terramaster seemed like a more powerful option on paper – even though it was a brand I wasn't familiar with. A little research later, and I decided to take the leap! 

However, I did uncover conflicting information during my research: while Terramaster advertises the F4-424 Pro with 32 GB of memory, Intel's [specifications](https://www.intel.com/content/www/us/en/products/sku/231805/intel-core-i3n305-processor-6m-cache-up-to-3-80-ghz/specifications.html) for the i3-N305 processor officially support a maximum of 16 GB. This raises questions about the actual memory configuration and potential limitations.


## Operating System
Unlike Synology, which locks you into their Disk Management System (DSM), TerraMaster gives me the freedom to choose my preferred operating system. I value this flexibility, especially after Synology's recent [decision](https://www.synology.com/en-us/releaseNote/StorageManager) to remove S.M.A.R.T. attribute recording. I prefer to maintain control over my systems and their features.

Given this freedom, I plan to experiment with different operating systems, including TOS, Unraid, FreeNAS, Proxmox, and possibly Ubuntu and Debian if the others don't meet my expectations. My current server runs Ubuntu Server 20.04 LTS, which has been rock-solid for its purpose. Ideally, I'm looking for an OS that's easy to maintain and control while still offering the features I need to support my projects.

### TOS
TerraMaster ships the F4-424 Pro with their own TerraMaster OS (TOS). Out of curiosity, I'll give it a brief test drive and perhaps run some benchmarks to compare it with other operating systems.

[Website](https://www.terra-master.com/global/alltos/)

### Unraid OS
Unraid is a new discovery for me. While it's not free, it offers a 30-day trial, which I plan to utilize for evaluation and potential benchmarking.

[Website](https://unraid.net/product)

### FreeNAS
FreeNAS enjoys a strong reputation. Their latest product, FreeNAS Scale, is Linux-based and includes a ZFS implementation with ARC memory allocation similar to TrueNAS Core, which I'm eager to try.

[Website](https://www.truenas.com/docs/scale/24.04/gettingstarted/scalereleasenotes/#24042-changelog)

### Proxmox
Proxmox is an open-source virtualization platform based on Debian. It provides essential tools for managing containers, virtual machines, and more. Since I have never used it, there's some learning curve to get ahold of it.

[Website](https://www.proxmox.com/en/proxmox-virtual-environment/features)

### Ubuntu 24.04
The latest LTS version of Ubuntu comes with a recent systemd version that includes helpful tools for secure boot and TPM features. I'm interested in exploring how these work and assessing systemd's progress.

[Website](https://ubuntu.com/server)

### Debian
If none of the above options prove satisfactory, I can always fall back on Debian. It's been my go-to for desktops over the past decade and has been consistently reliable. The downside of Debian is the increased maintenance effort required for services. Additionally, Debian's approach of locking distros to specific package versions, with only security and bug fixes, might make it less convenient to upgrade to newer packages at the OS level. However, this won't be a major issue for me, as most of my services will run in containers.

[Website](https://www.debian.org/News/2023/20230610)


#### Footnotes
[^fn1]: The i3-N305 does not have performance-cores, instead it has efficiency-cores which are based on [Gracemont microarchitecture](https://en.wikipedia.org/wiki/Gracemont_(microarchitecture)). One of the drawbacks is that E-cores does not have hyper-threading. [more information](https://www.assured-systems.com/faq/what-is-the-difference-between-p-core-and-e-core-processors/)
