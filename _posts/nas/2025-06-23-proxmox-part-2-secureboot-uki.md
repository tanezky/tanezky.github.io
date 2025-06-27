---
title: Proxmox Part 2 - Secure Boot & UKI
description: Working towards secure boot and Unified Kernel Image

date: 2025-06-23 20:00:00 +/-TTTT
author: tanezky

categories: [NAS]
tags: [NAS,Terramaster,F4-424 Pro, Proxmox, LUK, UKI, UEFI, Secure Boot]
image: /assets/img/posts/pve-part2-sb/pve-part2-post.png
---

## Motivation

This is Part 2 in my [Proxmox Project](/posts/proxmox-part-0-project-specification/#2-configure-secureboot) where I want to build secure Proxmox installation.

Part 2 will focus on using custom keys for Secure Boot and combining kernel into signed unified-kernel-image (UKI). After the Secure Boot is working, we'll remove the boot partition and cleanup the installation.

## A Brief Introduction to Secure Boot

Secure Boot is a security feature that prevents the execution of unauthorized code during the system boot process. It verifies the digital signature of each executable against a set of trusted keys stored within the UEFI firmware. By default, the UEFI contains a set of pre-installed public keys, which are a Platform Key (PK), Key Exchange Key (KEK), Signature Database (db), and Forbidden Signature Database (dbx).

![Secureboot signing chain](/assets/img/posts/pve-part2-sb/sb-keys.drawio.png){: width="641" height="61" .center}
_Key chain_

- **Platform Key (PK)** establishes a trust relationship between the platform owner and the platform's firmware.
- **Key Exhange Key (KEK)** is used to update the db and dbx databases.
- **Signature Database (db)** Contains authorized certificates; the boot executables must be signed with a key existing in this database.
- **Forbidden Signature Database (dbx)** Contains revoked certificates and signatures, [more information](https://uefi.org/revocationlistfile).

#### Why using custom keys?

An [inspection of the F4-424 Pro's UEFI](https://tanezky.github.io/posts/terramaster-uefi/) revealed the use of a default AMI test key, which has been publicly leaked. This indicates that TerraMaster did not replace the default key with an unique, vendor-specific key. Therefore, we will deviate from the standard approach and replace the existing keys with our own. While this will introduce some additional complexity in terms of future maintenance and updates, it significantly enhances the overall security of the installation. 


## Unified Kernel Image (UKI)

A typical Linux installation, such as the default Debian setup, consists of a bootloader, an initial RAM disk (initrd), and the kernel itself. During system startup, the BIOS/UEFI first executes the bootloader (e.g. GRUB) which in turn presents kernel command-line options and loads the initrd. The initrd plays a crucial role in preparing the system by loading necessary kernel modules and performing preliminary tasks before handing the control to the init system (e.g. systemd). 

In practice, an Unified Kernel Image (UKI) is a single, self-contained EFI executable that encapsulates a EFI stub program, the kernel, initrd, and command-line options. Using UKI has a several advantages. 

Firstly, it eliminates the need for a separate bootloader which simplifies the boot process and reduces the potential attack surface.

Secondly, typically in Secure Boot setups the initrd often remains unsigned and unverified. This presents a potential security vulnerability where an attacker could physically remove the hard drive, connect it to another system, access the unencrypted boot partition, extract the initrd, inject malicious code, and then repackage the initrd. Upon reinserting the drive and booting the system, the compromised initrd would execute the malicious code, potentially leading to further security breaches, such as printing the encryption key passed by TPM during the boot.


## Preparing Dev Environment

### Encrypted Storage

Since I'm going to work on security sensitive data, I will use the Samsung FIT USB drive as an encrypted storage. The drive is connected to the internal USB port and serves as a backup location in case the boot process breaks and I have to reinstall the whole system.


```shell
# 1. Use the whole USB drive as an encrypted device (/dev/sda)
cryptsetup luksFormat --type luks2 /dev/sda

# 2. Open newly created LUKS device as /dev/mapper/samsungfit
cryptsetup luksOpen /dev/sda samsungfit

# 3. Create filesystem
mkfs -t ext4 -V /dev/mapper/samsungfit

# 4. Mount the filesystem (creates mount folder if it doesn't exist)
mount -m /dev/mapper/samsungfit /media/samsungfit
```

### Clone the repo

Easiest way to follow the next steps in this article is to clone Project repo.

```shell
# Install git
apt install git

# Clone the repo inside the encrypted storage
git clone https://github.com/tanezky/proxmox-project.git /media/samsungfit/proxmox-project

# All the article related scripts and files are under part2 folder.
```

There is a helper script "init_ws_storage.sh" to manage with mounting the storage.

Simply copy it to your root user folder and modify the Variables in the beginning of the file according to yours.

Usage: `bash init_ws_storage.sh`

#### Without repo

Otherwise, copy files from the repo and create the structure manually

```shell
encrypted_storage
├── sb
│   └── sb_create_keys.sh
└── uki
    ├── splash.bmp
    └── uki_create.sh
```

### VS Code

I like to use VSCode as an editor instead of using terminal text editors. To achieve this I will use Natizyskunk's [SFTP extension](https://marketplace.visualstudio.com/items?itemName=Natizyskunk.sftp) to synchronize files between machines. I'm using encrypted container mounted as a workspace on my desktop PC to keep everything secure.

To configure SFTP connection, first give ownership of the files on remote machine to your non-root user via `chown -R tanezky:tanezky /media/samsungfit`

Create sftp config in VS Code `ctrl + shift p >  SFTP: Config`

sftp.json config file (obtain agent with `echo ${SSH_AUTH_SOCK}`):

```json 
{
    "name": "pve-vault",
    "host": "10.42.42.150",
    "username": "tanezky",
    "protocol": "sftp",
    "remotePath": "/media/samsungfit",
    "uploadOnSave": true,
    "useTempFile": true,
    "openSsh": true,
    "agent": "/run/user/1000/keyring/ssh"
}
```


## Secure Boot

First, it's good to check current status of the Secure Boot
```shell
root@pve-vault:~# mokutil --sb-state
SecureBoot disabled
root@pve-vault:~# dmesg | grep Secure
[    0.000000] secureboot: Secure boot disabled
[    0.013585] secureboot: Secure boot disabled
```

### Create keys

Now that we have a secure place for working on sensitive content, it's time to create keys for secure boot.

The [script](https://github.com/tanezky/proxmox-project/blob/main/part2/sb/sb_create_keys.sh) is located in `proxmox-project/part2/sb`

It will ask password to create keys and it will again ask passwords to sign auth files.

It is important to use strong passwords for the keys. Store them in secure location, for example in a password manager.


```shell
# First, install dependencies
apt install efitools

# Run the script
sh sb_create_keys.sh

# Once the script has been succesfully executed, it has organized files into subdirectories
sb
├── auth
│   ├── db.auth
│   ├── KEK.auth
│   └── PK.auth
├── certs
│   ├── db.crt
│   ├── KEK.crt
│   └── PK.crt
├── esl
│   ├── db.esl
│   ├── KEK.esl
│   └── PK.esl
├── GUID.txt
└── keys
    ├── db.key
    ├── KEK.key
    └── PK.key

# Copy esl files on EFI partition, these will be enrolled in later stage once the UKI is created.
cp -r sb/esl /boot/efi/EFI
```

## Generating and Signing UKI

The [script](https://github.com/tanezky/proxmox-project/blob/main/part2/uki/uki_create.sh) to create UKI is located in `proxmox-project/part2/uki`

```shell
# Install dependencies
apt install systemd-boot-efi sbsigntool

# Run the script
bash uki_create.sh
```

When signing the UKI, you will be prompted for a password for previously generated db.key

The script will
- Find latest PVE kernel and initrd from `/boot`
- Get os-release information from `/usr/lib/os-release`
- Generate kernel cmdline options (more info below)
- Add boot splash image
- Create unsigned UKI
- Sign UKI with db.key and db.crt
- Move the signed UKI to location: `/boot/efi/EFI/pve/pve-uki.efi`

### What are the cmdline options used

The script will extract ROOT information from `/proc/cmdline`

Then it will be concatenated with `ro quiet splash`

`ro` and `quiet` are defaults used by Proxmox installations

`splash` is to show the boot splash image

Since Proxmox kernel versions 6.8 the `intel_iommu=on` is enabled by default, so there is no need to add it anymore.


## Creating Boot Entry

Next, we need to create boot entry for newly created UKI.

First, find the location of EFI partition on disk, it's where the `/boot/efi` is mounted.
```bash
root@pve-vault:~# lsblk

NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                   8:0    1  59.8G  0 disk  
└─samsungfit        252:1    0  59.7G  0 crypt /media/samsungfit
nvme0n1             259:0    0 465.8G  0 disk  
├─nvme0n1p1         259:1    0   487M  0 part  /boot/efi
├─nvme0n1p2         259:2    0   447G  0 part  
│ └─nvme0n1p2_crypt 252:0    0   447G  0 crypt /
└─nvme0n1p3         259:3    0  18.2G  0 part  /boot
```
On my device the EFI is first partition on the disk `nvme0n1`

```shell
# Check what are current boot entries (with details)
root@pve-vault:~# efibootmgr -v
BootCurrent: 0003
Timeout: 20 seconds
BootOrder: 0003
Boot0000* proxmox	VenHw(99e275e7-75a0-4b37-a2e6-c5385e6c00cb)
Boot0003* debian	HD(1,GPT,59abc397-fd4d-46d9-911c-a132743e35ea,0x800,0xf3800)/File(\EFI\debian\shimx64.efi)

# Create new entry
efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Proxmox SB" -l "\EFI\pve\pve-uki.efi"

# Check the boot entry configuration
efibootmgr -v

# Verify the path to efi binary is correct
....../File(\EFI\pve\pve-uki.efi)
```

## Enroll keys to UEFI and enable Secure Boot

{% include embed/youtube.html id='Di2-ndSh9TQ' %}


Reboot the device to UEFI `systemctl reboot --firmware-setup`

After setting up keys and enabling secure boot, save changes.

### Manual steps
1. Go to `Security -> Secure Boot`
2. Select `Reset to Setup Mode` -> Choose `Yes` -> Choose `No`
3. Go to `Key Management`
4. Choose `Platform Key`
5. Choose `Update`
6. Choose `No`
7. Select File System
8. Navigate to `EFI -> esl`
9. Choose `PK.esl`
10. Choose Public Key Certificate
11. Choose `Yes`
12. Confirm `Success`
13. Select `No` to "Reset without saving?"
14. Repeat steps to `Key Exhange Keys (KEK.esl)` and `Authorized Signatures (db.esl)`

Then, use ESC to return to Secure Boot menu and change Secure Boot to `Enabled`

Finally return with ESC, go to `Save & Exit` tab and choose `Save Changes and Exit`

## Verify Secure Boot 

Let's check the status of the Secure Boot
```shell
root@pve-vault:~# mokutil --sb-state 
SecureBoot enabled
root@pve-vault:~# dmesg | grep Secure
[    0.000000] secureboot: Secure boot enabled
[    0.000000] Kernel is locked down from EFI Secure Boot mode; see man kernel_lockdown.7
[    0.012983] secureboot: Secure boot enabled
```
Now we have succesfully enrolled our own keys for Secure Boot, created a signed unified-kernel-image and enabled the Secure boot.

## Removing /boot partition
![Partition layout](/assets/img/posts/pve-part2-sb/pve-part2-partition-layout.png){: width="815" height="291" .center}

Since we have now working Secure Boot and UKI, which is located on EFI partition, there's no need for separate `boot` partition anymore. In fact, to make the system more secure from prying eyes, we can now move the content from `boot` partition to encrypted root partition.

```shell
# Get information on partitions involved with /boot
root@pve-vault:~# lsblk
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                   8:0    1  59.8G  0 disk  
└─samsungfit        252:1    0  59.7G  0 crypt /media/samsungfit
nvme0n1             259:0    0 465.8G  0 disk  
├─nvme0n1p1         259:1    0   487M  0 part  /boot/efi
├─nvme0n1p2         259:2    0   447G  0 part  
│ └─nvme0n1p2_crypt 252:0    0   447G  0 crypt /
└─nvme0n1p3         259:3    0  18.2G  0 part  /boot

# We are interested in /boot/efi and /boot

# Unmount /boot/efi
umount /boot/efi

# Unmount /boot
umount /boot

# Mount boot partition to temporary location
mount -m /dev/nvme0n1p3 /mnt/boot

# Verify /boot/efi and /boot is not mounted anymore
root@pve-vault:~# lsblk
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                   8:0    1  59.8G  0 disk  
└─samsungfit        252:1    0  59.7G  0 crypt /media/samsungfit
nvme0n1             259:0    0 465.8G  0 disk  
├─nvme0n1p1         259:1    0   487M  0 part  
├─nvme0n1p2         259:2    0   447G  0 part  
│ └─nvme0n1p2_crypt 252:0    0   447G  0 crypt /
└─nvme0n1p3         259:3    0  18.2G  0 part  /mnt/boot

# Copy all files and links from /mnt/boot into /boot
cp -av /mnt/boot/. /boot/

# Verify all files got copied
ls -al /boot

# Remove unnecessary lost+found folder which came from boot partition
rm -rf /boot/lost+found

# Unmount boot partition
umount /mnt/boot
```

Next, we'll have to modify `/etc/fstab` and remove reference to the boot partition
```shell
root@pve-vault:~# cat /etc/fstab 
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# systemd generates mount units based on this file, see systemd.mount(5).
# Please run 'systemctl daemon-reload' after making changes here.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/mapper/nvme0n1p2_crypt /               ext4    errors=remount-ro 0       1
# /boot was on /dev/nvme0n1p3 during installation
UUID=e9245dfa-3a64-4a1c-a07d-b191d55ec4ff /boot           ext4    defaults        0       2
# /boot/efi was on /dev/nvme0n1p1 during installation
UUID=462A-1651  /boot/efi       vfat    umask=0077      0       1


#
# Remove boot references to make the /etc/fstab to look:
root@pve-vault:~# cat /etc/fstab 
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# systemd generates mount units based on this file, see systemd.mount(5).
# Please run 'systemctl daemon-reload' after making changes here.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/mapper/nvme0n1p2_crypt /               ext4    errors=remount-ro 0       1
# /boot/efi was on /dev/nvme0n1p1 during installation
UUID=462A-1651  /boot/efi       vfat    umask=0077      0       1
```

Then, reload systemd, mount the `/boot/efi` and verify it has content
```shell
# Reload systemd
systemctl daemon-reload

# Mount /boot/efi
mount /boot/efi

# Verify it has content
ls -al /boot/efi

# There should be EFI folder found
```

Before rebooting, it's good practice to update initramfs and recreate the UKI
```shell
# Recreate initramfs
update-initramfs -u -k all

# Then recreate the UKI
root@pve-vault:~# cd /media/samsungfit/proxmox-project/part2/uki/
root@pve-vault:/media/samsungfit/proxmox-project/part2/uki# bash uki_create.sh

# Reboot the system
reboot
```

## Cleaning

### Destroy boot partition
```shell
# After reboot, confirm boot partition is not mounted
root@pve-vault:~# lsblk
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                   8:0    1  59.8G  0 disk  
nvme0n1             259:0    0 465.8G  0 disk  
├─nvme0n1p1         259:1    0   487M  0 part  /boot/efi
├─nvme0n1p2         259:2    0   447G  0 part  
│ └─nvme0n1p2_crypt 252:0    0   447G  0 crypt /
└─nvme0n1p3         259:3    0  18.2G  0 part 

# Destroy contents of boot partition via overwriting it with random data (this will take some time depending on size of the partition)
dd if=/dev/urandom of=/dev/nvme0n1p3 bs=4M status=progress

# Delete the partition
fdisk /dev/nvme0n1
  p # list partitions
  d # delete partition
  3 # partition number (for nvme0n1p3)
  w # write and exit
```

### Remove SB certificates and old boot entries
```shell
# Remove Secure Boot certs from EFI partition
rm -rf /boot/efi/EFI/esl/

# View UEFI boot entries
root@pve-vault:~# efibootmgr -v
BootCurrent: 0001
Timeout: 20 seconds
BootOrder: 0001,0002
Boot0000* proxmox	VenHw(99e275e7-75a0-4b37-a2e6-c5385e6c00cb)
Boot0001* Proxmox SB	HD(1,GPT,59abc397-fd4d-46d9-911c-a132743e35ea,0x800,0xf3800)/File(\EFI\pve\pve-uki.efi)
Boot0002* debian	HD(1,GPT,59abc397-fd4d-46d9-911c-a132743e35ea,0x800,0xf3800)/File(\EFI\debian\shimx64.efi)..BO
Boot0003* debian	HD(1,GPT,59abc397-fd4d-46d9-911c-a132743e35ea,0x800,0xf3800)/File(\EFI\debian\shimx64.efi)

# Remove old entries (all but Proxmox SB)
efibootmgr -b 0000 -B
efibootmgr -b 0002 -B
efibootmgr -b 0003 -B

# Verify Proxmox SB is the only one existing
BootOrder: 0001
Boot0001* Proxmox SB	HD(1,GPT,59abc397-fd4d-46d9-911c-a132743e35ea,0x800,0xf3800)/File(\EFI\pve\pve-uki.efi)
```

In next article the goal is to initialize TPM2, use it to open LUKS partition when booting up the system.

## Sources

Following pages have been helpful while doing research on Secure Boot

- [Debian wiki - SecureBoot](https://wiki.debian.org/SecureBoot)
- [Debian wiki - EFIStub](https://wiki.debian.org/EFIStub)
- [Rod Smith's Managing EFI Boot Loaders for Linux](https://www.rodsbooks.com/efi-bootloaders/)
- [There's a Hole in the Boot](https://eclypsium.com/blog/theres-a-hole-in-the-boot/)
- [The Real Shim Shady - How CVE-2023-40547 Impacts Most Linux Systems](https://eclypsium.com/blog/the-real-shim-shady-how-cve-2023-40547-impacts-most-linux-systems/)

