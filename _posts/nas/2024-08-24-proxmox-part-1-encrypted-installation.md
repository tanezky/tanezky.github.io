---
title: Proxmox Part 1 - Encrypted Installation
description: Encrypted Proxmox installation on top of clean Debian 12

date: 2024-08-24 20:00:00 +/-TTTT
author: tanezky

categories: [NAS]
tags: [NAS,Terramaster,F4-424 Pro, Proxmox, SSH, LUKS]
image: /assets/img/posts/pve-part1-installation/pve-part1-installation-post.jpg
---

> This post has been rewritten in Jun 2025
{: .prompt-info }

## Motivation

If not already familiar with the goals, get familiar with the article Proxmox Part 0 - Secure Architecture

This article will mostly follow the official Proxmox [tutorial](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_12_Bookworm) on installing Proxmox on a fresh Debian 12.11 installation. Main difference being Debian configured with encryption.

## Installing Debian
![Debian Install](/assets/img/posts/pve-part1-installation/pve-part1-installation01.jpg){: width="1920" height="1080" .center}
1. [Download](https://www.debian.org/distrib/netinst) the Debian amd64 small installation image and flash it onto a USB thumb drive.
2. Boot your system from the USB drive and begin the installation process.
3. When the "Installation Menu" appears, **interrupt the process!**


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

The boot partition will not be encrypted, otherwise UEFI would not be able to execute the boot process. This is okay since we are not going to store any sensitive data on the partition.

Create the following partitions:

- EFI Partition: 512 MB
    - **Use as:** EFI System Partition.
- Boot Partition: 2 GB
    - **Use as:** ext4
    - **Format the partition:** yes,format it
    - **Mount point:** `/boot`
    - **Mount options:** `noatime`
- Encrypted Volume: 480 GB
    - **Use as:** physical volume for encryption
    - **Erase data:** no (this disk has never had anything sensitive on it)

Use the screenshot above as an example before configuring the encryption


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

### Getting error "cryptsetup: Waiting for encrypted source device"
![cryptsetup: Waiting for encrypted source device error](/assets/img/posts/pve-part1-installation/pve-part1-cryptsetup_error.jpg)
This error appears when reinstalling the system without formatting drives, the bootloader tries to run the configuration from previous installation. As a workaround, either enter to the `Advanced options for Debian GNU/Linux` from GRUB menu and choose other kernel or, alternatively reinstall the system and ensure EFI and boot partitions are formatted during installation.

## Configuring SSH for first connection

It's more convenient to continue the installation from a remote machine, and SSH is the perfect tool for this. However, SSH employs a [TOFU](https://en.wikipedia.org/wiki/Trust_on_first_use) (Trust on First Use) authentication scheme, requiring us to verify the connection when connecting to an unknown endpoint for the first time. Furthermore, the default OpenSSH server configuration relies on password authentication which is vulnerable to man-in-the-middle attacks.

While it's unlikely my network has any malicious actors waiting to exploit my devices, I try to prioritize security best practices whenever possible.

Following steps instructs on creating encrypted vault on an USB stick to store SSH-keys.

Log in as root on the Debian machine:
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

# 6. Create an empty file for encrypted container (50 MB)
dd if=/dev/urandom of=pvekey.img bs=1M count=50

# 7. Create LUKS volume within the empty file
cryptsetup --verify-passphrase luksFormat pvekey.img

# 8. Open the LUKS volume and confirm it is visible in the system
cryptsetup open --type luks pvekey.img keyvault
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
ssh-keygen -t ed25519 -f pve_key

# 13. Create the SSH configuration folder for the non-admin user
mkdir /home/netzky/.ssh

# 14. Add the public key to the user's authorized_keys file
cat pve_key.pub >> /home/netzky/.ssh/authorized_keys

# 15. Give the user permissions to .ssh folder and files
chown -R netzky:netzky /home/netzky/.ssh/

# 16. Finally, exit the folder and unmount drive.
cd /
umount /media/keyvault
cryptsetup close keyvault
umount /mnt/usb
```

### First SSH Connection
![First SSH Connection](/assets/img/posts/pve-part1-installation/pve-part1-installation08.jpg){: width="821" height="582" .center}

Connect the USB drive to your PC and move `pvekey.img` on your hard drive.

On my Gnome Desktop I can mount the filevault via double clicking the file.
After entering the password it is mounted.

How-to in shell:

```shell
# 1. Open the LUKS container
sudo cryptsetup open --type luks pvekey.img keyvault

# 2. If not existing, create location for the mount point
sudo mkdir -p /media/keyvault

# 3. Mount
sudo mount /dev/mapper/keyvault /media/keyvault
```

Finally, connect to the device with SSH

```bash
# 1. Navigate to the mounted location or open a new terminal in that folder
cd /media/keyvault

# 2. List files, there should be pve_key and vault-fingerprint.txt
ls
lost+found  vault-fingerprint.txt  pve_key  pve_key.pub


# 3. Print contents of vault-fingerprint.txt
cat vault-fingerprint.txt
256 SHA256:wXkuNcQt/8W7N2ivAwJctaYq5mVncEmIrj574mfyYto root@pve-vault (ED25519)

# 4. Create SSH connection using pve_key
ssh -i pve_key netzky@10.42.42.150
The authenticity of host '10.42.42.150 (10.42.42.150)' can't be established.'
ED25519 key fingerprint is SHA256:wXkuNcQt/8W7N2ivAwJctaYq5mVncEmIrj574mfyYto.
This key is not known by any other names.

# 5. Copy paste the SHA256 fingerprint from step 3
Are you sure you want to continue connecting (yes/no/[fingerprint])? SHA256:wXkuNcQt/8W7N2ivAwJctaYq5mVncEmIrj574mfyYto
Warning: Permanently added '10.42.42.150' (ED25519) to the list of known hosts.

# 6. Enter password for the pve_key
Enter passphrase for key 'pve_key': 

# 7. Login complete
Linux pve-vault 6.1.0-37-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.99-1 (2024-07-15) x86_64

The programs included with the Debian GNU/Linux system are free software;
...
```
#### Make SSH easier
Typing the full ssh command with the key file every time can get tedious. To streamline this, it's a good idea to create a configuration file that simplifies future connections.

```shell
# First, copy the private key in your .ssh folder
cp pve_key /home/netzky/.ssh/

# Edit the local SSH config file on your PC
~/.ssh/config

# Add following content with your details
Host    pve-vault
        HostName 10.42.42.150
        User netzky
        IdentityFile ~/.ssh/pve_key
        LogLevel INFO
        Compression yes

# Making connection is now much more straightforward
ssh pve-vault
```

## Installing Proxmox

### Preparation
Before installing the Proxmox, it is good to list all currently installed packages. This can be used for comparison later to see what new packages was installed.
```shell
# Save list of installed packages
dpkg --get-selections > packages_before_proxmox.txt
# Save detailed list of installed packages
apt list --installed > packages_before_proxmox_detailed.txt
# Save list of manually installed packages
apt-mark showmanual > manually_installed_before_proxmox.txt
```


Ensure `/etc/hosts` maps `127.0.1.1` only to localhost

```shell
# Become root, enter root password which was given during debian installation
su -

# Check /etc/hosts to be similar as below, edit if necessary
cat /etc/hosts
127.0.0.1	localhost
10.42.42.150	pve-vault.proxmox.com pve-vault

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Verify `hostname --ip-address` returns the IP-address assigned during the Debian installation
```shell
root@pve-vault:~# hostname --ip-address 
10.42.42.150
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
![First SSH Connection](/assets/img/posts/pve-part1-installation/pve-part1-installation09.jpg){: width="1202" height="648" .center}
_For Postfix Configuration I chose No configuration since I don't have a mail server (yet)._


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

Before adding network bridge, let's create new lists of installed packages
```shell
# Save list of installed packages
dpkg --get-selections > packages_after_proxmox.txt
# Save detailed list of installed packages
apt list --installed > packages_after_proxmox_detailed.txt
# Save list of manually installed packages
apt-mark showmanual > manually_installed_after_proxmox.txt
# Comparison can be done with
diff -u file_before file_after
```

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
- 29.12.2024 - Update Postfix Configuration, add reboot instruction to cleanup.
- 23.06.2025 - Whole post was reviewed and updated