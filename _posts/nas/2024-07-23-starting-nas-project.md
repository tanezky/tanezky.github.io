---
title: Planning - NAS Project & Requirements
author: tanezky
date: 2024-07-23 17:00:00 +/-TTTT
description: Beginning of NAS Project
categories: [NAS]
tags: [nas]
image: /assets/img/posts/project_post.jpg
---

## Motivation

My trusty old i5 desktop PC has served as my homelab server faithfully for over a decade. However, it's time for an upgrade. I'm looking for something more energy-efficient, with a smaller footprint that better suits my apartment space. Additionally, the current storage is insufficient for my family's growing needs for file syncing and disaster recovery.

## Requirements

I had a few key requirements in mind for the NAS:

- **Processor** A multi-core processor with virtualization support to handle multiple tasks and potentially run virtual machines.
- **Memory** At least 16 GB of RAM to ensure smooth operation of services and leave some room for additional experimentation in my homelab.
- **Capacity** A minimum of 16 TB of usable disk space to accommodate current and future storage needs.
- **Redundancy** Disk failure protection (software zRAID)
- **Power Consumption** Good power efficiency to reduce energy usage and lower my electricity bill.
- **Security** Support for FTPM (Firmware Trusted Platform Module) to ensure secure boot and protect against unauthorized firmware modifications.
**Size** A compact form factor, smaller than ATX.
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

I'd been considering the [Synology DiskStation DS923+](), but the Terramaster seemed like a more powerful option on paper – even though it was a brand I wasn't familiar with. A little research later, and I decided to take the leap! 

However, I did uncover conflicting information during my research: while Terramaster advertises the F4-424 Pro with 32 GB of memory, Intel's [specifications](https://www.intel.com/content/www/us/en/products/sku/231805/intel-core-i3n305-processor-6m-cache-up-to-3-80-ghz/specifications.html) for the i3-N305 processor officially support a maximum of 16 GB. This raises questions about the actual memory configuration and potential limitations.

[^fn1]: The i3-N305 does not have performance-cores, instead it has efficiency-cores which are based on [Gracemont microarchitecture](https://en.wikipedia.org/wiki/Gracemont_(microarchitecture)). One of the drawbacks is that E-cores does not have hyper-threading. [more information](https://www.assured-systems.com/faq/what-is-the-difference-between-p-core-and-e-core-processors/)