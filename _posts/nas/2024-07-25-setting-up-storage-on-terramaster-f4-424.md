---
title: Choosing and Installing the Storage
author: tanezky
date: 2024-07-25 20:00:00 +/-TTTT
description: A Storage strategy and disk installation on Terramaster F4-424 Pro NAS device
categories: [NAS]
tags: [NAS,Terramaster,F4-424 Pro]
image: /assets/img/posts/storage-setup-terramaster-f4-424-pro/storage_post.jpg
---


## Requirements
I had a few key requirements in mind for storage

- **Capacity:** A minimum of 16 TB of usable disk space to accommodate current and future storage needs.
- **Redundancy:** Disk failure protection (software RAID-Z)
- **User Quotas:** Enough storage quota for 8 users, with a minimum of 1-2TB per user.
- **Future Expandability:** The ability to expand into a geographically redundant setup [(3-2-1)](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/).

## Storage Strategy
![Zpool](/assets/img/posts/storage-setup-terramaster-f4-424-pro/disk-layout.png){: width="409" height="444" .w-75 .center}

After some research, I've decided to give ZFS a try. The native encryption, on-the-fly compression, and snapshotting features are too compelling to ignore. However, as I continue testing the NAS, this storage strategy might evolve.

My initial plan is to utilize four drives for storage, organized into two vdevs (virtual devices), with each vdev comprising a mirrored pair of hard drives (similar to RAID 10) within a single zpool. This configuration prioritizes performance and redundancy. Each vdev can operate independently, effectively doubling potential read and write speeds compared to a single RAIDZ1 configuration. Additionally, if one drive fails within a vdev, the other drive maintains data accessibility, and the pool can be rebuilt (resilvered) without downtime.

For the operating system, services, and other data that needs quick access, I'll dedicate one NVMe drive. ZFS won't be necessary for this drive, as support for OS installation on ZFS varies across Linux distributions.


## HDD Research
![Zpool](/assets/img/posts/storage-setup-terramaster-f4-424-pro/Traditional-HD-vs-SMR-HD.jpg){: width="1156" height="604" .w-75 .center}
_CMR vs SMR. Source: Buffalo Americas [^fn1]_

Given the high storage capacity requirements for my homelab, NVMe or SSDs were quickly ruled out due to cost. After revisiting different storage solutions and delving into the world of hard disks, it became clear that Conventional Magnetic Recording (CMR) was the way to go, avoiding Shingled Magnetic Recording (SMR) at all costs.

To understand why, let's look at how these technologies work. High-capacity hard drives consist of stacked platters, with more platters equating to more capacity. Each platter has tracks for data storage. On CMR disks, there are gaps between these tracks, while on SMR drives, the tracks overlap, increasing data density. This allows for higher capacity disks with fewer platters, ultimately lowering manufacturing costs and making SMR drives cheaper than CMR.  

However, SMR comes with a significant speed penalty. Due to the overlapping tracks, SMR drives first store data in a temporary location and later write it to its final, overlapping position. This leads to considerably slower transfer speeds compared to CMR, especially for constant access scenarios where the drive lacks time to reorganize data. In a NAS environment, where parity is typically used across disks, the resilvering process (rebuilding data after a drive failure) can become excruciatingly slow with SMR drives.[^fn2] While SMR may be suitable for archival purposes where data access is infrequent[^fn1], its performance limitations make it unsuitable for a NAS device.


## Choosing HDD
With the storage technology (CMR) decided, it was time to select the specific hard drives. My criteria were straightforward: CMR technology, designed for NAS usage, positive user reviews, and avoiding refurbished disks.

After extensive research, I narrowed down my options to four potential HDD lineups: WD Red Plus, WD Red Pro, Seagate Ironwolf, and Toshiba N300. WD Red Plus and Pro exceeded my budget, while the Toshiba N300 had concerning customer reviews citing failures within months of use.

Seagate Ironwolf seemed like a strong contender, with good reviews and widespread recommendations. While searching for deals on Ironwolf disks, I stumbled upon a great offer for the Seagate Exos X18 16 TB. Despite being slightly pricier than Ironwolf, its enterprise-grade features and specifications convinced me to invest in it.


## NVMe for OS
My positive experiences with Samsung NVMe drives in my desktop computer led me to choose the Samsung V-NAND SSD 980 for the NAS. I opted for the 500 GB model, the most affordable PCIe 3.0 option that still delivers sufficient read and write speeds for the operating system and essential services.

While marketed as "3bit MLC," which is effectively TLC (triple-level cell) technology, Samsung offers a 5-year warranty and a generous 300 TBW (terabytes written) rating. This seemed a good choice for my NAS usage, and I'm optimistic about its longevity.


## USB
For the internal USB connector, I picked up a thumb drive to use primarily for testing utilities and additional storage during experimentation. I went for the smallest option available, a Samsung FIT Plus with 64GB capacity. The datasheet indicates it can withstand operating temperatures up to 60°C, which should be enough for my needs.


## Overview of Disks

| Port | Device | Datasheet |
| ---- | ------ | --------- |
| USB2 | Samsung FIT Plus | [Link](/assets/files/datasheets/samsung-fit-plus-datasheet.pdf) |
| SATA1 | Seagate Exos X18 | [Link](/assets/files/datasheets/exos-x18-channel-DS2045-1-2007GB-en_SG.pdf) |
| SATA2 | Seagate Exos X18 | [Link](/assets/files/datasheets/exos-x18-channel-DS2045-1-2007GB-en_SG.pdf) |
| SATA3 | Seagate Exos X18 | [Link](/assets/files/datasheets/exos-x18-channel-DS2045-1-2007GB-en_SG.pdf) |
| SATA4 | Seagate Exos X18 | [Link](/assets/files/datasheets/exos-x18-channel-DS2045-1-2007GB-en_SG.pdf) |
| M.2_SSD_1 | Samsung SSD980 | [Link](/assets/files/datasheets/Samsung_NVMe_SSD_980_Data_Sheet_Rev.1.1.pdf) |
| M.2_SSD_2 | Empty | |

## Installing Disks and Observations

### USB
The space allotted for the USB thumb drive is remarkably tight, requiring the removal of a motherboard support leg to fit the Samsung thumb drive. Even then, clearance is minimal, and a longer thumb drive would simply not fit.

![Drive in place](/assets/img/posts/storage-setup-terramaster-f4-424-pro/storage02.jpg){: width="1848" height="1947" .w-75 .center}
_Closer look, there's no space left_

![Measuring Terramaster USB](/assets/img/posts/storage-setup-terramaster-f4-424-pro/storage03.jpg){: width="2484" height="1661" .w-75 .center}
_Terramaster USB thumb drive is 18.19 mm long_

![Drive in place](/assets/img/posts/storage-setup-terramaster-f4-424-pro/storage04.jpg){: width="2670" height="1715" .w-75 .center}
_Samsung FIT USB thumb drive is 23.62 mm long_

### Hard Drives
![Installation instruction](/assets/img/posts/storage-setup-terramaster-f4-424-pro/storage05.jpg){: width="1200" height="561" .w-75 .center}
_Terramaster HDD installation instruction_

The hard drives are installed without screws, and once in place, a small gap remains between them, allowing for airflow from the front to the back of the device.

![Closer look to hdd gaps](/assets/img/posts/storage-setup-terramaster-f4-424-pro/storage06.jpg){: width="3365" height="1353" .w-75 .center}
_Closer look to gaps between HDDs_

![NAS from frontside](/assets/img/posts/storage-setup-terramaster-f4-424-pro/storage07.jpg){: width="1704" height="1612" .w-75 .center}
_HDDs installed_

### Air Circulation and CPU Heatsink

![HHD in bay 4](/assets/img/posts/storage-setup-terramaster-f4-424-pro/storage08.jpg){: width="2387" height="1848" .w-75 .center}
_HDD installed in bay 4_

The proximity of the hard drive in bay 4 to the CPU heatsink raises concerns about potential heat buildup. The ventilation holes underneath the enclosure might not provide sufficient airflow, and the 25mm thick fan at the back might not adequately cool the heatsink, especially under heavy load. A thinner 15mm fan could potentially improve airflow and cooling in this area but it is unlikely to find any slimmer models with equivalent airflow specifications. The effectiveness of the current cooling solution and the potential for overheating will require monitoring during operation, particularly under heavy workloads.

![25mm fan in back](/assets/img/posts/storage-setup-terramaster-f4-424-pro/storage09.jpg){: width="3062" height="1527" .w-75 .center}
_25mm fan in back_

![Ventilation holes for motherboard](/assets/img/posts/storage-setup-terramaster-f4-424-pro/storage10.jpg){: width="3645" height="1847" .w-75 .center}
_Ventilation holes for motherboard_

![Ventilation holes under drive bays](/assets/img/posts/storage-setup-terramaster-f4-424-pro/storage11.jpg){: width="4000" height="1848" .w-75 .center}
_Ventilation holes under drive bays_

![Ventilation holes under the device](/assets/img/posts/storage-setup-terramaster-f4-424-pro/storage12.jpg){: width="2548" height="1680" .w-75 .center}
_Ventilation holes under the device_


#### Footnotes

[^fn1]: [Buffalo Americaas: CMR vs SMR](https://www.buffalotech.com/resources/cmr-vs-smr-hard-drives-in-network-attached-storage-nas-msp)
[^fn2]: [ServeTheHome: WD Red SMR vs CMR Performance testing](https://www.servethehome.com/wd-red-smr-vs-cmr-tested-avoid-red-smr/2/)
