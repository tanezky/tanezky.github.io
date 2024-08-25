---
title: Proxmox - Cheatsheet
author: tanezky
date: 2024-08-17 21:00:00 +/-TTTT
description: Nice-to-know commands to manage Proxmox
categories: [NAS]
tags: [NAS,Terramaster,F4-424 Pro, Proxmox, Cheatsheet]
image: /assets/img/posts/pve/pve-cheatsheet-post.jpg
---

> This article will be updated while experimenting on Proxmox.
{: .prompt-tip }

## Cheatsheet for Proxmox
List of commands I might need to use again some day.

[PVE Official Documentation](https://pve.proxmox.com/pve-docs/)

### Basic Linux Log Commands
```shell
# Observe for new dmesg messages
dmesg -W

# Follow journal log
journalctl --follow
```


### qm - Virtual Machine
```shell
vmid=100

# Config
/etc/pve/qemu-server/${vmid}.conf

# Start and observe VM terminal
qm start ${vmid} && sleep 1 && qm terminal ${vmid}

# Stop
qm stop ${vmid}

# VM Locked, cannot stop
qm unlock ${vmid}

# Forcely stop VM (last resort, not recommended)
ps aux | grep "/usr/bin/kvm -id ${vmid}"
kill -9 PID
```
Sources: [qm cmd](https://pve.proxmox.com/pve-docs/qm.1.html), [qm.conf](https://pve.proxmox.com/wiki/Manual:_qm.conf), [qm PCI Passthrough](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_pci_passthrough), 

### lscpi - List PCI device details
```shell
# List devices and their vendor id information
lspci -nn

# List devices and which kernel module and driver is in use
lspci -k

# List device capabilities identified by system
lspci -vv
```

### PCI device control
```shell
# Query power state
cat /sys/bus/pci/devices/0000\:03\:00.0/power_state 
D0

# Query reset state
cat /sys/bus/pci/devices/0000\:03\:00.0/reset_method 
pm bus

# Query If d3Cold allowed
cat /sys/bus/pci/devices/0000\:03\:00.0/d3cold_allowed
1

# Remove PCI device
echo 1 > /sys/bus/pci/devices/0000\:03\:00.0/remove

# Initiate scan, re-adds removed PCI device if it is not in failed state
echo 1 > /sys/bus/pci/rescan
```

### LUKS
#### Create Vault
```shell
# Create an empty file for encrypted container (512 MB)
dd if=/dev/urandom of=filevault.img bs=1M count=512

# Create LUKS volume within the empty file
cryptsetup --verify-passphrase luksFormat filevault.img

# Open the LUKS volume
cryptsetup open --type luks filevault.img keyvault

# Create filesystem labeled as keyvault
mkfs.ext4 -L keyvault /dev/mapper/keyvault

# Mount
mount /dev/mapper/keyvault /media/keyvault
```

#### Controlling LUKS Vault
```shell
# Open
cryptsetup open --type luks vaultfile preferred-name

# Location of opened vaults
/dev/mapper

# Close, unmount before closing
umount /media/mountpoint
cryptsetup close preferred-name
```


### Other Sources
- General
    - [Proxmox VE Helper-Scripts](https://tteck.github.io/Proxmox/)
- Kernel
    - [The kernel’s command-line parameters](https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html)
    - [Proxmox VE Kernel](https://pve.proxmox.com/wiki/Proxmox_VE_Kernel)
- QEMU
    - [Qemu-guest-agent](https://pve.proxmox.com/wiki/Qemu-guest-agent)
- General Passthrough
    - [Passthrough Physical Disk to Virtual Machine (VM)](https://pve.proxmox.com/wiki/Passthrough_Physical_Disk_to_Virtual_Machine_(VM))
- PCI Passthrough & IOMMU
    - [PCI Passthrough](https://pve.proxmox.com/wiki/PCI_Passthrough)
    - [Blog: IOMMU Groups, inside and out](https://vfio.blogspot.com/2014/08/iommu-groups-inside-and-out.html)
    - [PCI passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)





