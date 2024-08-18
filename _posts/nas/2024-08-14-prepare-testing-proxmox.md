---
title: Proxmox VE 8.2.4
author: tanezky
date: 2024-08-14 20:00:00 +/-TTTT
description: Preparations to test Proxmox Virtual Environment
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

## Enable IOMMU
An Input-Output Memory Management Unit (IOMMU) is a hardware component that improves memory management for peripheral devices. It acts as a bridge, translating device-specific addresses into physical memory addresses for seamless communication between I/O devices and system memory. This translation enhances security by isolating devices and preventing them from accessing unauthorized memory locations. Furthermore, IOMMU significantly improves virtualization by preventing processes from accessing overlapping memory areas.

```shell
# Verify if IOMMU is enabled, look for DMAR: IOMMU enabled
dmesg | grep -e DMAR -e IOMMU
[    0.011417] ACPI: DMAR 0x0000000073178000 000088 (v02 INTEL  EDK2     00000002      01000013)
[    0.011459] ACPI: Reserving DMAR table memory at [mem 0x73178000-0x73178087]
[    0.170330] DMAR: Host address width 39
[    0.170332] DMAR: DRHD base: 0x000000fed90000 flags: 0x0
[    0.170342] DMAR: dmar0: reg_base_addr fed90000 ver 4:0 cap 1c0000c40660462 ecap 29a00f0505e
[    0.170345] DMAR: DRHD base: 0x000000fed91000 flags: 0x1
[    0.170350] DMAR: dmar1: reg_base_addr fed91000 ver 5:0 cap d2008c40660462 ecap f050da
[    0.170352] DMAR: RMRR base: 0x0000007c000000 end: 0x000000803fffff
[    0.170356] DMAR-IR: IOAPIC id 2 under DRHD base  0xfed91000 IOMMU 1
[    0.170358] DMAR-IR: HPET id 0 under DRHD base 0xfed91000
[    0.170359] DMAR-IR: Queued invalidation will be enabled to support x2apic and Intr-remapping.
[    0.171493] DMAR-IR: Enabled IRQ remapping in x2apic mode
[    0.409441] pci 0000:00:02.0: DMAR: Skip IOMMU disabling for graphics
[    0.481654] DMAR: No ATSR found
[    0.481655] DMAR: No SATC found
[    0.481657] DMAR: IOMMU feature fl1gp_support inconsistent
[    0.481658] DMAR: IOMMU feature pgsel_inv inconsistent
[    0.481659] DMAR: IOMMU feature nwfs inconsistent
[    0.481660] DMAR: IOMMU feature dit inconsistent
[    0.481660] DMAR: IOMMU feature sc_support inconsistent
[    0.481661] DMAR: IOMMU feature dev_iotlb_support inconsistent
[    0.481662] DMAR: dmar0: Using Queued invalidation
[    0.481666] DMAR: dmar1: Using Queued invalidation
[    0.483816] DMAR: Intel(R) Virtualization Technology for Directed I/O

# Modify /etc/default/grub and add kernel parameter
nano /etc/default/grub
# Modify
GRUB_CMDLINE_LINUX=""
# to:
GRUB_CMDLINE_LINUX="intel_iommu=on"
# Update grub
update-grub2


# Ensure kernel modules are loaded
nano modules-load.d/iommu-vfio.conf
# Add following to the file
vfio
vfio_iommu_type1
vfio_pci

# Update initramfs and reboot
update-initramfs -u -k all

# Verify DMAR: IOMMU enabled is found from dmesg
dmesg | grep -e DMAR -e IOMMU
[    0.011708] ACPI: DMAR 0x0000000073178000 000088 (v02 INTEL  EDK2     00000002      01000013)
[    0.011750] ACPI: Reserving DMAR table memory at [mem 0x73178000-0x73178087]
[    0.066312] DMAR: IOMMU enabled
[    0.168660] DMAR: Host address width 39
[    0.168661] DMAR: DRHD base: 0x000000fed90000 flags: 0x0
[    0.168675] DMAR: dmar0: reg_base_addr fed90000 ver 4:0 cap 1c0000c40660462 ecap 29a00f0505e
[    0.168677] DMAR: DRHD base: 0x000000fed91000 flags: 0x1
[    0.168682] DMAR: dmar1: reg_base_addr fed91000 ver 5:0 cap d2008c40660462 ecap f050da
[    0.168685] DMAR: RMRR base: 0x0000007c000000 end: 0x000000803fffff
[    0.168688] DMAR-IR: IOAPIC id 2 under DRHD base  0xfed91000 IOMMU 1
[    0.168689] DMAR-IR: HPET id 0 under DRHD base 0xfed91000
[    0.168691] DMAR-IR: Queued invalidation will be enabled to support x2apic and Intr-remapping.
[    0.169817] DMAR-IR: Enabled IRQ remapping in x2apic mode
[    0.406220] pci 0000:00:02.0: DMAR: Skip IOMMU disabling for graphics
[    0.476989] DMAR: No ATSR found
[    0.476990] DMAR: No SATC found
[    0.476991] DMAR: IOMMU feature fl1gp_support inconsistent
[    0.476992] DMAR: IOMMU feature pgsel_inv inconsistent
[    0.476993] DMAR: IOMMU feature nwfs inconsistent
[    0.476994] DMAR: IOMMU feature dit inconsistent
[    0.476995] DMAR: IOMMU feature sc_support inconsistent
[    0.476996] DMAR: IOMMU feature dev_iotlb_support inconsistent
[    0.476997] DMAR: dmar0: Using Queued invalidation
[    0.477001] DMAR: dmar1: Using Queued invalidation
[    0.479106] DMAR: Intel(R) Virtualization Technology for Directed I/O
```