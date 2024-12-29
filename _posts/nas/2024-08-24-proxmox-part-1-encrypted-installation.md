---
title: Proxmox Part 1 - Encrypted Installation
description: Encrypted Proxmox installation on top of clean Debian 12

date: 2024-08-24 20:00:00 +/-TTTT
author: tanezky

categories: [NAS]
tags: [NAS,Terramaster,F4-424 Pro, Proxmox, SSH, LUKS]
image: /assets/img/posts/pve-part1-installation/pve-part1-installation-post.jpg
---

## Motivation
This article will closely follow the official Proxmox [tutorial](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_12_Bookworm) on installing Proxmox on a fresh Debian 12 installation.

## Installing Debian
![Debian Install](/assets/img/posts/pve-part1-installation/pve-part1-installation01.jpg){: width="1920" height="1080" .center}
1. Download the Debian installer ISO image and flash it onto a USB thumb drive.
2. Boot your system from the USB drive and begin the installation process.
3. When the "Installation Menu" appears, interrupt the process.


![Debian installer](/assets/img/posts/pve-part1-installation/pve-part1-installation02.jpg){: width="1920" height="1080" .center}

By default, the Debian installer attempts to use IP autoconfiguration and DHCP to obtain an IP address. We need to force it to use manual configuration instead.

- Press `e` to enter edit mode.
- Add `netcfg/disable_autoconfig=true` to the linux boot command line.
- Press `F10` to continue booting.

![Debian installer network](/assets/img/posts/pve-part1-installation/pve-part1-installation03.jpg){: width="1920" height="1080" .center}

Follow the installation process, choosing your preferred settings until you reach the Network Configuration step. Here, configure your network interface with a static IP address, netmask, gateway, and DNS server address. I use Cloudflare DNS `1.1.1.1` on my installations for improved performance.

![Debian installer partitioning](/assets/img/posts/pve-part1-installation/pve-part1-installation04.jpg){: width="1920" height="1080" .center}

Proceed with the installation until you reach the **Partition Disks** section. Select the **Manual** partitioning method.

Delete all existing partitions on the disk intended for Proxmox installation. If you encounter errors related to existing LVM configurations, navigate to **Configure the Logical Volume Manager** and delete the volumes and groups.

Create the following partitions:

- EFI Partition: 512 MB, **Use as:** EFI System Partition.
- Boot Partition: 2 GB, **Use as:** ext4, **Mount point:** `/boot` and **Mount options:** `noatime`
- Encrypted Volume: 480 GB, **Use as:** physical volume for encryption, **Erase data:** no (this disk has never had anything sensitive on it)

![Debian installer encryption](/assets/img/posts/pve-part1-installation/pve-part1-installation05.jpg){: width="1920" height="1080" .center}

Enter the "Configure encrypted volumes" section.

1. Check "Yes" to write changes to disk.
2. Choose "Create encrypted volumes."
3. Select the crypto device.
4. On next screen, select "Finish."
5. Enter and confirm a strong encryption password.

![Debian installer finalisation](/assets/img/posts/pve-part1-installation/pve-part1-installation06.jpg){: width="1920" height="1080" .center}

Once the encrypted volume appears in the "Partition Disks" view, modify it:

- **Use as:** ext4
- **Mount point:** `/`
- **Mount options:** `noatime`

Finally:

- "Finish partitioning and write changes to disk."
- Choose "no" when asked about configuring swap.
- Choose "yes" to write the changes to the disk.

![Debian installer software](/assets/img/posts/pve-part1-installation/pve-part1-installation07.jpg){: width="1920" height="1080" .center}

For Software selection choose `SSH server` and `standard system utilities`.

Once installation is complete, reboot the system and remove installation media.

## Configuring SSH for first connection

It's more convenient to continue the installation from a remote machine, and SSH is the perfect tool for this. However, SSH employs a [TOFU](https://en.wikipedia.org/wiki/Trust_on_first_use) (Trust on First Use) authentication scheme, requiring us to verify the connection when connecting to an unknown endpoint for the first time. Furthermore, the default OpenSSH server configuration relies on password authentication which is vulnerable to man-in-the-middle attacks.

While it's unlikely my network has any malicious actors waiting to exploit my devices, I try to prioritize security best practices whenever possible.

Log in as root on the Debian machine we just installed:

```shell
# Prepare the USB drive: (Skip steps 2 and 3 if you're not using the Debian installer USB)
# 1. Identify the USB drive with two partitions (/dev/sdb in my case)
lsblk

# 2. Create a new partition on the USB
fdisk /dev/sdb
    n (new partition)
    p (primary)
    3 (next available partition number)
    Press Enter to accept the default first sector.
    Press Enter to use all remaining free space.
    w (write changes and exit)

# 3. Format the new partition
mkfs.ext4 /dev/sdb3

# 4. Create a mount point and mount the partition
mkdir -p /mnt/usb
mount /dev/sdb3 /mnt/usb

# 5. Navigate to the mounted drive
cd /mnt/usb

# 6. Create an empty file for encrypted container (512 MB)
dd if=/dev/urandom of=filevault.img bs=1M count=512

# 7. Create LUKS volume within the empty file
cryptsetup --verify-passphrase luksFormat filevault.img

# 8. Open the LUKS volume and confirm it is visible in the system
cryptsetup open --type luks filevault.img keyvault
ls /dev/mapper

# 9. Create filesystem labeled as keyvault
mkfs.ext4 -L keyvault /dev/mapper/keyvault

# 10. Create mount point for the keyvault, mount it and navigate to the mounted folder
mkdir -p /media/keyvault
mount /dev/mapper/keyvault /media/keyvault
cd /media/keyvault

# 11. Retrieve and save the host key fingerprint
ssh-keygen -l -f /etc/ssh/ssh_host_ed25519_key >> host_fingerprint.txt

# 12. Generate SSH keys (Use a strong password)
ssh-keygen -t ed25519 -f vault_key

# 13. Create the SSH configuration folder for the non-admin user
mkdir /home/tanezky/.ssh

# 14. Copy the public key to the user's authorized_keys file
cat vault_key.pub >> /home/tanezky/.ssh/authorized_keys

# 15. Give the user permissions to .ssh folder and files
chown -R tanezky:tanezky /home/tanezky/.ssh/

# 16. Finally, exit the folder and unmount drive.
cd /
umount /media/keyvault
cryptsetup close keyvault
umount /mnt/usb
```

### First SSH Connection
![First SSH Connection](/assets/img/posts/pve-part1-installation/pve-part1-installation08.jpg){: width="1920" height="1080" .center}

Connect the USB drive to your PC and move `filevault.img` on your hard drive.

On my Gnome Desktop I can mount the filevault via double clicking the file.
After entering the password it is mounted.

How-to in shell:

```shell
# 1. Open the LUKS container
sudo cryptsetup open --type luks filevault.img keyvault

# 2. If not existing, create location for the mount point
sudo mkdir -p /media/keyvault

# 3. Mount
sudo mount /dev/mapper/keyvault /media/keyvault
```

Finally, connect to the device with SSH

```bash
# 1. Navigate to the mounted location or open a new terminal in that folder
cd /media/keyvault

# 2. List files, there should be vault_key and vault-fingerprint.txt
ls
lost+found  vault-fingerprint.txt  vault_key  vault_key.pub


# 3. Print contents of vault-fingerprint.txt
cat vault-fingerprint.txt
256 SHA256:h+YtATVdtGddY5c3hCM9Negu+rS/pKrpliXI/Lg7s2U root@aurum-vault (ED25519)

# 4. Create SSH connection using vault_key
ssh -i vault_key tanezky@10.42.42.150
The authenticity of host '10.42.42.150 (10.42.42.150)' can't be established.'
ED25519 key fingerprint is SHA256:h+YtATVdtGddY5c3hCM9Negu+rS/pKrpliXI/Lg7s2U.
This key is not known by any other names.

# 5. Copy paste the SHA256 fingerprint from step 3
Are you sure you want to continue connecting (yes/no/[fingerprint])? SHA256:h+YtATVdtGddY5c3hCM9Negu+rS/pKrpliXI/Lg7s2U
Warning: Permanently added '10.42.42.150' (ED25519) to the list of known hosts.

# 6. Enter password for the vault_key
Enter passphrase for key 'vault_key': 

# 7. Login complete
Linux aurum-vault 6.1.0-23-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.99-1 (2024-07-15) x86_64

The programs included with the Debian GNU/Linux system are free software;
...
```
#### Make SSH easier
Typing the full ssh command with the key file every time can get tedious. To streamline this, it's a good idea to create a configuration file that simplifies future connections.

```shell
# Edit the local SSH config file on your PC
~/.ssh/config

# Add following content with your details
Host    vault
        HostName 10.42.42.150
        User tanezky
        IdentityFile /media/keyvault/vault_key
        LogLevel INFO
        Compression yes

# Making connection is now much more straightforward
ssh vault
```

## Installing Proxmox
Ensure `/etc/hosts` does not have `127.0.1.1` in it and the hostname is allocated with IP address

```shell
# Become root, enter root password which was given during debian installation
su -

# Check /etc/hosts to be similar as below, edit if necessary
cat /etc/hosts
127.0.0.1	localhost
10.42.42.150	aurum-vault.proxmox.com aurum-vault

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
#### Repository
Add the Proxmox VE repository to apt sources

```shell
echo "deb [arch=amd64] http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list
```

Add Proxmox VE repository key

```shell
wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg

# Verify checksum of the repository key
echo "7da6fe34168adc6e479327ba517796d4702fa2f8b4f0a9833f5ea6e6b48f6507a6da403a274fe201595edc86a84463d50383d07f64bdde2e3658108db7d6dc87 /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg" | sha512sum -c

# Should output:
/etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg: OK
```
#### Sources and Packages
Update sources, run full-upgrade, install Proxmox VE kernel and reboot

```shell
apt update && apt full-upgrade -y && apt install -y proxmox-default-kernel && systemctl reboot
```

After reboot install Proxmox VE packages

```shell
apt install -y proxmox-ve postfix open-iscsi chrony
```
![First SSH Connection](/assets/img/posts/pve-part1-installation/pve-part1-installation09.jpg){: width="1278" height="930" .center}
_For Postfix Configuration I chose Local only since I don't have a mail server (yet)._


#### Cleanup
After reboot, remove Debian kernel, unnecessary packages, update grub and remove subscription repository

```shell
# Remove kernel and unnecessary packages
apt remove -y linux-image-amd64 'linux-image-6.1*' os-prober

# Update grub and check its config
update-grub

# Remove pve-enterprise.list (don't remove unless you have subscription)
rm /etc/apt/sources.list.d/pve-enterprise.list
```

#### Finalising Setup

Last thing on the list is to add network bridge, for time being I'm going to use default configuration.

More information on Proxmox Network configuration on their [wiki](https://pve.proxmox.com/wiki/Network_Configuration#_default_configuration_using_a_bridge) page.

```shell
# Change /etc/network/interfaces from this:
....

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto enp2s0
iface enp2s0 inet static
    address 10.42.42.150/24
    gateway 10.42.42.1
    dns-nameservers 1.1.1.1
# dns-* options are implemented by the resolvconf package, if installed

iface enp1s0 inet manual



# To look something like this:
...

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

iface enp2s0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.42.42.150/24
    gateway 10.42.42.1
    bridge-ports enp2s0
    bridge-stp off
    bridge-fd 0
# dns-* options are implemented by the resolvconf package, if installed

iface enp1s0 inet manual



# Reboot the system after the bridge configuration is created
systemctl reboot

```

## Admin Web-interface
![Finalise Setup](/assets/img/posts/pve-part1-installation/pve-part1-installation10.jpg){: width="1278" height="930" .center}

Installation is complete, it's time to start using the new setup.

Go to admin web-interface <https://10.42.42.150:8006>

Use your `root` credentials and **Realm:** Linux PAM standard authentication

#### Page updates
29.12.2024 - Update Postfix Configuration, add reboot instruction to cleanup.