---
title: unRAID
author: tanezky
date: 2024-08-12 20:00:00 +/-TTTT
description: Time to take unRAID 7.0.0-beta.2 for a quick spin
categories: [NAS]
tags: [NAS,Terramaster,F4-424 Pro, unRAID]
image: /assets/img/posts/unraid/unraid-post.jpg
---

> **Disclaimer:** I only experimented with unRAID very briefly, so take this article with a grain of salt.
{: .prompt-info }

## unRAID
During my research for NAS operating systems, unRAID was frequently recommended. So, I decided to give it a quick spin.

unRAID is a proprietary Slackware Linux-based OS developed by Lime Technology Inc. Unlike traditional installations, the OS resides on a USB drive. The unRAID license is tied to the USB flash drive's GUID, meaning it won't function on drives without one.

unRAID's capabilities are categorized into three core areas:

- **Software-defined NAS** Sharing storage over the network.
- **Application server** Running Docker containers.
- **Hardware virtualization** Running virtual machines.

The latest stable version is unRAID 6.12.11, which I'll install first and then upgrade to 7.0.0-beta.2. While 6.12.11 supports ZFS pools, it enforces the use of Array Devices, which isn't ideal for my NVMe drive since Array Devices lacks TRIM support. The upcoming unRAID 7 removes this limitation, allowing for standalone ZFS pools. Additionally, unRAID 7 has significant improvements to its ZFS implementation. If I were to consider paying for unRAID, it would be for version 7, at least on paper.

The Uncast Show has a video about upcoming Unraid 7

{% include embed/youtube.html id='f2VwlAw5xXU' %}

## Updating to 7.0.0-beta.2
![Upgrade prompt](/assets/img/posts/unraid/unraid01.jpg){: width="1665" height="1090" .center}
_Upgrading to unRAID 7_
After flashing the USB drive with the unRAID flashing tool, I booted the system, configured my credentials, opted for the trial period, and switched to the black theme to save my eyes.  Then it was time to update the system.

`Tools -> Update OS -> Next -> View Changelog to Update -> Continue Update on.. --> Confirm and start update`

## Setting up ZFS
![Setup ZFS](/assets/img/posts/unraid/unraid02.jpg){: width="1665" height="1264" .center}
_Setup ZFS Pools on Main page_

1. Set Array Devices Slots to: none
2. Add 2 pools, first named `nvme-zfs` with 1 slots, second named `vault-zfs` with 4 slots
3. Set NVMe drive to frist slot
4. Enter to `nvme-zfs` configuration page (screenshot under `nmve-zfs Configuration Page`)
5. Set HDDs for the `vault-zfs` pool
6. Enter to `vault-zfs` configuration from first link (screenshot under `vault-zfs Configuration Page`)
7. Set passphrase for the encrypted zfs
8. Start disk pools

### nmve-zfs Configuration Page
![nvme-zfs config](/assets/img/posts/unraid/unraid03.jpg){: width="1670" height="1232" .center}
1. Filesystem type: zfs - encrypted
2. Enable compression
3. Leave autotrim on
4. Apply settings
5. Return to Main

### vault-zfs Configuration Page
![vault-zfs config](/assets/img/posts/unraid/unraid04.jpg){: width="1670" height="1232" .center}
1. Filesystem type: zfs - encrypted
2. Allocation profile: mirror, 2 groups of 2 devices
3. Enable compression
4. Disable autotrim
5. Apply settings
6. Return to Main page

### Format Disks
![zfs unmountable:unformatted](/assets/img/posts/unraid/unraid05.jpg){: width="1667" height="819" .center}
Disk pools have been started but needs to be formatted before disks can be mounted, an error "Unmountable: unformatted" is presented.
1. Confirm format of the disks, read the warning in shown popup dialog
2. Format

Once formatting is complete unRAID mounts disks.

## Network Shares
![zfs unmountable:unformatted](/assets/img/posts/unraid/unraid06.jpg){: width="1665" height="588" .center}
Enable NFS under `Settings -> NFS`

For testing purposes, I created public shares without authentication. I'll be assessing transfer speeds within my 1Gbps network, which is likely to be the limiting factor here.

```shell
# Find out on unRAID server path to nfs shares
root@Tower:~# cat /etc/exports

# See exports(5) for a description.
# This file contains a list of all directories exported to other computers.
# It is used by rpc.nfsd and rpc.mountd.
"/mnt/user/nvme-share" -fsid=102,async,no_subtree_check *(rw,sec=sys,insecure,anongid=100,anonuid=99,all_squash)
"/mnt/user/vault-share" -fsid=103,async,no_subtree_check *(rw,sec=sys,insecure,anongid=100,anonuid=99,all_squash)

# Mount nfs shares on desktop PC
$ sudo mount -t nfs 10.42.42.150:/mnt/user/nvme-share /opt/nfs-nvme-share
$ sudo mount -t nfs 10.42.42.150:/mnt/user/vault-share /opt/nfs-vault-share
```

### Transfer speeds on nfs-nvme-share
```shell
# Writing 35GB with dd
$ dd if=/dev/zero of=/opt/nfs-nvme-share/35G bs=1M count=35840 conv=fdatasync status=progress

37564186624 bytes (38 GB, 35 GiB) copied, 301 s, 125 MB/s37580963840 bytes (38 GB, 35 GiB) copied, 301.114 s, 125 MB/s
35840+0 records in
35840+0 records out
37580963840 bytes (38 GB, 35 GiB) copied, 320.725 s, 117 MB/s

# Reading 35GB file from remote
$ dd if=/opt/nfs-nvme-share/35G  of=/dev/null bs=1M status=progress

37486592000 bytes (37 GB, 35 GiB) copied, 309 s, 121 MB/s
35840+0 records in
35840+0 records out
37580963840 bytes (38 GB, 35 GiB) copied, 309.806 s, 121 MB/s

# Writing ubuntu-24.04 image with rsync
$ rsync --stats --progress -h ubuntu-24.04-desktop-amd64.iso /opt/nfs-nvme-share/

6.11G 100%  170.48MB/s    0:00:34 (xfr#1, to-chk=0/1)
Number of files: 1 (reg: 1)
Number of created files: 1 (reg: 1)
Number of deleted files: 0
Number of regular files transferred: 1
Total file size: 6.11G bytes
Total transferred file size: 6.11G bytes
Literal data: 6.11G bytes
Matched data: 0 bytes
File list size: 0
File list generation time: 0.001 seconds
File list transfer time: 0.000 seconds
Total bytes sent: 6.12G
Total bytes received: 35

sent 6.12G bytes  received 35 bytes  104.55M bytes/sec
total size is 6.11G  speedup is 1.00

# Reading ubuntu-24.04 image with rsync
$ rsync --stats --progress -h /opt/nfs-nvme-share/ubuntu-24.04-desktop-amd64.iso .

6.11G 100%  104.71MB/s    0:00:55 (xfr#1, to-chk=0/1)
Number of files: 1 (reg: 1)
Number of created files: 1 (reg: 1)
Number of deleted files: 0
Number of regular files transferred: 1
Total file size: 6.11G bytes
Total transferred file size: 6.11G bytes
Literal data: 6.11G bytes
Matched data: 0 bytes
File list size: 0
File list generation time: 0.001 seconds
File list transfer time: 0.000 seconds
Total bytes sent: 6.12G
Total bytes received: 35

sent 6.12G bytes  received 35 bytes  108.25M bytes/sec
total size is 6.11G  speedup is 1.00

```
![transfer speeds1](/assets/img/posts/unraid/unraid07.jpg){: width="812" height="221" .center}
_RAM utilisation while testing NVMe shares_

Overall CPU load was ~7% during write and ~10% during read.
I restarted the device between write and read to clear ZFS cache.


### Transfer speeds on nfs-vault-share
```shell
# Writing 35GB with dd
$ dd if=/dev/zero of=/opt/nfs-vault-share/35G bs=1M count=35840 conv=fdatasync status=progress

37547409408 bytes (38 GB, 35 GiB) copied, 297 s, 126 MB/s37580963840 bytes (38 GB, 35 GiB) copied, 297.292 s, 126 MB/s
35840+0 records in
35840+0 records out
37580963840 bytes (38 GB, 35 GiB) copied, 320.994 s, 117 MB/s


# Reading 35GB file from remote
$ dd if=/opt/nfs-vault-share/35G  of=/dev/null bs=1M status=progress

37549506560 bytes (38 GB, 35 GiB) copied, 324 s, 116 MB/s
35840+0 records in
35840+0 records out
37580963840 bytes (38 GB, 35 GiB) copied, 324.276 s, 116 MB/s


# Writing ubuntu-24.04 image with rsync
$ rsync --stats --progress -h ubuntu-24.04-desktop-amd64.iso /opt/nfs-vault-share/

6.11G 100%  185.20MB/s    0:00:31 (xfr#1, to-chk=0/1)
Number of files: 1 (reg: 1)
Number of created files: 1 (reg: 1)
Number of deleted files: 0
Number of regular files transferred: 1
Total file size: 6.11G bytes
Total transferred file size: 6.11G bytes
Literal data: 6.11G bytes
Matched data: 0 bytes
File list size: 0
File list generation time: 0.001 seconds
File list transfer time: 0.000 seconds
Total bytes sent: 6.12G
Total bytes received: 35

sent 6.12G bytes  received 35 bytes  104.55M bytes/sec
total size is 6.11G  speedup is 1.00


# Reading ubuntu-24.04 image with rsync
$ rsync --stats --progress -h /opt/nfs-vault-share/ubuntu-24.04-desktop-amd64.iso .

6.11G 100%  105.81MB/s    0:00:55 (xfr#1, to-chk=0/1)
Number of files: 1 (reg: 1)
Number of created files: 1 (reg: 1)
Number of deleted files: 0
Number of regular files transferred: 1
Total file size: 6.11G bytes
Total transferred file size: 6.11G bytes
Literal data: 6.11G bytes
Matched data: 0 bytes
File list size: 0
File list generation time: 0.002 seconds
File list transfer time: 0.000 seconds
Total bytes sent: 6.12G
Total bytes received: 35

sent 6.12G bytes  received 35 bytes  110.20M bytes/sec
total size is 6.11G  speedup is 1.00
```
![transfer speeds2](/assets/img/posts/unraid/unraid08.jpg){: width="812" height="220" .center}
_RAM utilisation while testing HDD shares_
Overall CPU load was ~9% during write and ~12% during read.
I restarted the device between write and read to clear ZFS cache.

## Testing throughput with iperf3
![iperf3 image](/assets/img/posts/unraid/unraid09.jpg){: width="1665" height="436" .center}
_iperf3 Docker image_

Iperf tests were ran from my desktop PC, acting as a client.

### TCP Tests
```shell
# Run basic TCP test
$ iperf3 -c 10.42.42.150
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  1.10 GBytes   943 Mbits/sec    0   sender
[  5]   0.00-10.00  sec  1.10 GBytes   941 Mbits/sec        receiver


# Run basic TCP test with 10 parallel streams
$ iperf3 -c 10.42.42.150 -P 10
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   113 MBytes  94.8 Mbits/sec    0             sender
[  5]   0.00-10.01  sec   112 MBytes  93.6 Mbits/sec                  receiver
[  7]   0.00-10.00  sec   114 MBytes  95.3 Mbits/sec    0             sender
[  7]   0.00-10.01  sec   112 MBytes  94.1 Mbits/sec                  receiver
[  9]   0.00-10.00  sec   114 MBytes  95.9 Mbits/sec    0             sender
[  9]   0.00-10.01  sec   113 MBytes  95.0 Mbits/sec                  receiver
[ 11]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec    0             sender
[ 11]   0.00-10.01  sec   113 MBytes  94.4 Mbits/sec                  receiver
[ 13]   0.00-10.00  sec   114 MBytes  95.4 Mbits/sec    0             sender
[ 13]   0.00-10.01  sec   113 MBytes  94.4 Mbits/sec                  receiver
[ 15]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec    0             sender
[ 15]   0.00-10.01  sec   113 MBytes  94.3 Mbits/sec                  receiver
[ 17]   0.00-10.00  sec   113 MBytes  95.1 Mbits/sec    0             sender
[ 17]   0.00-10.01  sec   113 MBytes  94.3 Mbits/sec                  receiver
[ 19]   0.00-10.00  sec   112 MBytes  93.6 Mbits/sec    0             sender
[ 19]   0.00-10.01  sec   110 MBytes  92.4 Mbits/sec                  receiver
[ 21]   0.00-10.00  sec   114 MBytes  95.3 Mbits/sec    0             sender
[ 21]   0.00-10.01  sec   113 MBytes  94.3 Mbits/sec                  receiver
[ 23]   0.00-10.00  sec   114 MBytes  95.2 Mbits/sec    0             sender
[ 23]   0.00-10.01  sec   113 MBytes  94.3 Mbits/sec                  receiver
[SUM]   0.00-10.00  sec  1.11 GBytes   952 Mbits/sec    0             sender
[SUM]   0.00-10.01  sec  1.10 GBytes   941 Mbits/sec                  receiver


# Run basic TCP test with 10 parallel streams in reverse
$ iperf3 -c 10.42.42.150 -P 10 -R
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   114 MBytes  95.3 Mbits/sec    0             sender
[  5]   0.00-10.00  sec   112 MBytes  94.2 Mbits/sec                  receiver
[  7]   0.00-10.00  sec   114 MBytes  95.3 Mbits/sec    0             sender
[  7]   0.00-10.00  sec   112 MBytes  94.1 Mbits/sec                  receiver
[  9]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec    0             sender
[  9]   0.00-10.00  sec   112 MBytes  94.1 Mbits/sec                  receiver
[ 11]   0.00-10.00  sec   113 MBytes  95.2 Mbits/sec   31             sender
[ 11]   0.00-10.00  sec   112 MBytes  94.1 Mbits/sec                  receiver
[ 13]   0.00-10.00  sec   114 MBytes  95.3 Mbits/sec   34             sender
[ 13]   0.00-10.00  sec   112 MBytes  94.1 Mbits/sec                  receiver
[ 15]   0.00-10.00  sec   113 MBytes  94.7 Mbits/sec    0             sender
[ 15]   0.00-10.00  sec   112 MBytes  94.0 Mbits/sec                  receiver
[ 17]   0.00-10.00  sec   113 MBytes  94.9 Mbits/sec    0             sender
[ 17]   0.00-10.00  sec   112 MBytes  94.1 Mbits/sec                  receiver
[ 19]   0.00-10.00  sec   113 MBytes  94.8 Mbits/sec    0             sender
[ 19]   0.00-10.00  sec   112 MBytes  94.0 Mbits/sec                  receiver
[ 21]   0.00-10.00  sec   113 MBytes  94.8 Mbits/sec    0             sender
[ 21]   0.00-10.00  sec   112 MBytes  94.0 Mbits/sec                  receiver
[ 23]   0.00-10.00  sec   113 MBytes  94.8 Mbits/sec    0             sender
[ 23]   0.00-10.00  sec   112 MBytes  94.1 Mbits/sec                  receiver
[SUM]   0.00-10.00  sec  1.11 GBytes   951 Mbits/sec   65             sender
[SUM]   0.00-10.00  sec  1.09 GBytes   940 Mbits/sec                  receiver
```

### UDP Tests
```shell
# Run UDP test with 1000 Mbit/sec connection
$ iperf3 -c 10.42.42.150 -u -b 1000M
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.00  sec  1.11 GBytes   956 Mbits/sec  0.000 ms  0/825559 (0%)  sender
[  5]   0.00-10.00  sec  1.11 GBytes   956 Mbits/sec  0.033 ms  0/825557 (0%)  receiver


# Run UDP test with 1000 Mbit/sec connection and with 10 parallel streams
$ iperf3 -c 10.42.42.150 -u -b 1000M -P 10
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec  0.000 ms  0/82640 (0%)  sender
[  5]   0.00-10.01  sec   114 MBytes  95.7 Mbits/sec  0.038 ms  0/82640 (0%)  receiver
[  7]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.000 ms  0/82566 (0%)  sender
[  7]   0.00-10.01  sec   114 MBytes  95.6 Mbits/sec  0.378 ms  0/82566 (0%)  receiver
[  9]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.000 ms  0/82568 (0%)  sender
[  9]   0.00-10.01  sec   114 MBytes  95.6 Mbits/sec  0.127 ms  0/82568 (0%)  receiver
[ 11]   0.00-10.00  sec   114 MBytes  95.8 Mbits/sec  0.000 ms  0/82748 (0%)  sender
[ 11]   0.00-10.01  sec   114 MBytes  95.8 Mbits/sec  0.304 ms  0/82748 (0%)  receiver
[ 13]   0.00-10.00  sec   114 MBytes  95.9 Mbits/sec  0.000 ms  0/82833 (0%)  sender
[ 13]   0.00-10.01  sec   114 MBytes  95.9 Mbits/sec  0.040 ms  0/82832 (0%)  receiver
[ 15]   0.00-10.00  sec   114 MBytes  95.2 Mbits/sec  0.000 ms  0/82194 (0%)  sender
[ 15]   0.00-10.01  sec   114 MBytes  95.1 Mbits/sec  0.041 ms  0/82194 (0%)  receiver
[ 17]   0.00-10.00  sec   114 MBytes  95.3 Mbits/sec  0.000 ms  0/82294 (0%)  sender
[ 17]   0.00-10.01  sec   114 MBytes  95.3 Mbits/sec  0.064 ms  0/82293 (0%)  receiver
[ 19]   0.00-10.00  sec   114 MBytes  96.0 Mbits/sec  0.000 ms  0/82899 (0%)  sender
[ 19]   0.00-10.01  sec   114 MBytes  96.0 Mbits/sec  0.215 ms  0/82899 (0%)  receiver
[ 21]   0.00-10.00  sec   115 MBytes  96.2 Mbits/sec  0.000 ms  0/83015 (0%)  sender
[ 21]   0.00-10.01  sec   115 MBytes  96.1 Mbits/sec  0.133 ms  0/83015 (0%)  receiver
[ 23]   0.00-10.00  sec   114 MBytes  95.5 Mbits/sec  0.000 ms  0/82446 (0%)  sender
[ 23]   0.00-10.01  sec   114 MBytes  95.4 Mbits/sec  0.298 ms  0/82446 (0%)  receiver
[SUM]   0.00-10.00  sec  1.11 GBytes   957 Mbits/sec  0.000 ms  0/826203 (0%)  sender
[SUM]   0.00-10.01  sec  1.11 GBytes   956 Mbits/sec  0.164 ms  0/826201 (0%)  receiver


# Run UDP test with 1000 Mbit/sec connection and with 10 parallel streams in reverse
$ iperf3 -c 10.42.42.150 -u -b 1000M -P 10 -R
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec  0.000 ms  0/0 (0%)  sender
[  5]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.039 ms  0/82561 (0%)  receiver
[  7]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec  0.000 ms  0/0 (0%)  sender
[  7]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.041 ms  0/82561 (0%)  receiver
[  9]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec  0.000 ms  0/0 (0%)  sender
[  9]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.051 ms  0/82561 (0%)  receiver
[ 11]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 11]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.045 ms  0/82560 (0%)  receiver
[ 13]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 13]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.041 ms  0/82561 (0%)  receiver
[ 15]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 15]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.049 ms  0/82560 (0%)  receiver
[ 17]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 17]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.050 ms  0/82560 (0%)  receiver
[ 19]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 19]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.054 ms  0/82560 (0%)  receiver
[ 21]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 21]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.046 ms  0/82559 (0%)  receiver
[ 23]   0.00-10.00  sec   114 MBytes  95.7 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 23]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.041 ms  0/82575 (0%)  receiver
[SUM]   0.00-10.00  sec  1.11 GBytes   957 Mbits/sec  0.000 ms  0/0 (0%)  sender
[SUM]   0.00-10.00  sec  1.11 GBytes   956 Mbits/sec  0.046 ms  0/825618 (0%)  receiver
```
## Power Consumption
At idle, the device consumed approximately 34 Watts. During file transfers the power consumption increased to 43 Watts.

## Conclusion
unRAID proved to be remarkably user-friendly for setting up ZFS pools and creating shares. It's easy to see why it's so widely recommended, and the extensive collection of community-provided applications is a major plus.

However, while unRAID supports ZFS encryption, the operating system disk itself remains unencrypted. Personally I have been using full-disk encryption for a long time and it is a crucial part of the security in case of theft or unauthorized access. This limitation in unRAID's current implementation is a significant drawback for me.