---
title: TrueNAS Scale
author: tanezky
date: 2024-08-13 20:00:00 +/-TTTT
description: Testing TrueNAS Scale for my NAS project
categories: [NAS]
tags: [NAS,Terramaster,F4-424 Pro, TrueNAS]
image: /assets/img/posts/truenas/truenas-post.jpg
---

> **Disclaimer:** I only experimented with TrueNAS very briefly, so take this article with a grain of salt.
{: .prompt-info }

## TrueNAS Scale
![TrueNAS Scale](/assets/img/posts/truenas/truenas01.jpg){: width="1665" height="1090" .center}
_TrueNAS Scale Dashboard_
TrueNAS, previously known as FreeNAS, is a system I explored a while back when I was testing various operating systems. Back then it was FreeBSD-based, which still exists, it has been rebranded as TrueNAS CORE. In July 2020, iXsystems, the company behind TrueNAS [announced](https://www.truenas.com/freenas/) TrueNAS SCALE which is a Linux-based variant.

The core feature of TrueNAS is its OpenZFS-based storage system which is accessible over the network. It also has built-in capabilities for running virtual machines and Kubernetes containers. TrueNAS has a marketplace for installing containerized applications, offering only iXsystems-verified apps by default. At the time of writing there were 107 such apps available. Additionally there's a community-driven platform called [TrueCharts](https://truecharts.org/) where users can find hundreds of additional applications. According to the [announcement](https://forums.truenas.com/t/the-future-of-electric-eel-and-apps/5409) from May, next TrueNAS Scale version [24.10 "Electric Eel"](https://www.truenas.com/docs/scale/gettingstarted/scalereleasenotes/) will have native Docker and Docker Compose support for apps.

For my testing I'll be using the latest TrueNAS version 24.04.2 "Dragonfish" installed on Samsung NVMe. This version includes OpenZFS 2.2.4, [introducing](https://github.com/openzfs/zfs/releases/tag/zfs-2.2.0) a fully adaptive ARC (Adaptive Replacement Cache). Previously the cache was statically set to 50% of total RAM on Linux based systems, but now it can dynamically adjust its size, using all available RAM when needed.

## Create ZFS Pool
![Manual Disk Selection](/assets/img/posts/truenas/truenas02.jpg){: width="1665" height="1111" .center}
_Manual Disk Selection_

Under `Storage --> Create Pool`

**General Info**
- Name: vault
- Encryption: Enabled
- Encryption Standard: AES-256-GCM 

**Data**
- Layout: Mirror
- Disk Size: 14.55 TiB (HDD)
- Width: 2
- Number of VDEVs: 2

Create pool and download the encryption key.

## Add Dataset
![Add Dataset](/assets/img/posts/truenas/truenas03.jpg){: width="1666" height="1109" .center}

Under `Datasets --> Add Dataset`

Give it a name, for example `vault-share` and save.
1. Make new dataset active
2. Scroll on right pane to Permissions and `Edit`

![Add Dataset](/assets/img/posts/truenas/truenas04.jpg){: width="1663" height="1107" .center}

> This example creates **unprotected** share for testing purposes.
{: .prompt-warning }

1. Add Write permission for `Group` and `Other`
2. Select `Apply permissions recurvisely`
3. Save

## Create NFS Share
![Add Share](/assets/img/posts/truenas/truenas05.jpg){: width="1477" height="906" .center}
_Under Shares --> UNIX (NFS Shares) --> Add_


## NFS Transfer Speeds
```shell
# Mount nfs shares on desktop PC
$ sudo mount -t nfs 10.42.42.150:/mnt/vault/vault-share /opt/nfs-vault-share
```


#### Transfer speeds on vault-share
```shell
# Writing 35GB with dd
$ dd if=/dev/zero of=/opt/nfs-vault-share/35G bs=1M count=35840 conv=fdatasync status=progress

37535875072 bytes (38 GB, 35 GiB) copied, 296 s, 127 MB/s37580963840 bytes (38 GB, 35 GiB) copied, 296.392 s, 127 MB/s
35840+0 records in
35840+0 records out
37580963840 bytes (38 GB, 35 GiB) copied, 321.135 s, 117 MB/s


# Reading 35GB file from remote
$ dd if=/opt/nfs-vault-share/35G  of=/dev/null bs=1M status=progress

37507563520 bytes (38 GB, 35 GiB) copied, 320 s, 117 MB/s
35840+0 records in
35840+0 records out
37580963840 bytes (38 GB, 35 GiB) copied, 320.621 s, 117 MB/s


# Writing ubuntu-24.04 image with rsync
$ rsync --stats --progress -h ubuntu-24.04-desktop-amd64.iso /opt/nfs-vault-share/

6.11G 100%  170.02MB/s    0:00:34 (xfr#1, to-chk=0/1)
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

sent 6.12G bytes  received 35 bytes  102.79M bytes/sec
total size is 6.11G  speedup is 1.00


# Reading ubuntu-24.04 image with rsync without reboot
$ rsync --stats --progress -h /opt/nfs-vault-share/ubuntu-24.04-desktop-amd64.iso .

6.11G 100%  103.44MB/s    0:00:56 (xfr#1, to-chk=0/1)
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


# Reading ubuntu-24.04 image with rsync after reboot
$ rsync --stats --progress -h /opt/nfs-vault-share/ubuntu-24.04-desktop-amd64.iso .

6.11G 100%  106.87MB/s    0:00:54 (xfr#1, to-chk=0/1)
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

sent 6.12G bytes  received 35 bytes  112.22M bytes/sec
total size is 6.11G  speedup is 1.00
```


## Testing Troughput with iperf3
![iperf on server](/assets/img/posts/truenas/truenas06.jpg){: width="1175" height="1111" .center}
TrueNAS had iperf3 already installed so running the tests was straightforward.

### TCP Tests
```shell
# Run basic TCP test
$ iperf3 -c 10.42.42.150
Connecting to host 10.42.42.150, port 5201
[  5] local 10.42.42.228 port 37432 connected to 10.42.42.150 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   114 MBytes   954 Mbits/sec    0    395 KBytes       
[  5]   1.00-2.00   sec   112 MBytes   943 Mbits/sec    0    395 KBytes       
[  5]   2.00-3.00   sec   112 MBytes   941 Mbits/sec    0    395 KBytes       
[  5]   3.00-4.00   sec   112 MBytes   940 Mbits/sec    0    414 KBytes       
[  5]   4.00-5.00   sec   113 MBytes   947 Mbits/sec    0    414 KBytes       
[  5]   5.00-6.00   sec   112 MBytes   940 Mbits/sec    0    414 KBytes       
[  5]   6.00-7.00   sec   112 MBytes   940 Mbits/sec    0    414 KBytes       
[  5]   7.00-8.00   sec   113 MBytes   945 Mbits/sec    0    414 KBytes       
[  5]   8.00-9.00   sec   112 MBytes   940 Mbits/sec    0    414 KBytes       
[  5]   9.00-10.00  sec   112 MBytes   939 Mbits/sec    0    414 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  1.10 GBytes   943 Mbits/sec    0             sender
[  5]   0.00-10.00  sec  1.10 GBytes   941 Mbits/sec                  receiver


# Run basic TCP test with 10 parallel streams
$ iperf3 -c 10.42.42.150 -P 10
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  51.6 MBytes  43.3 Mbits/sec    0             sender
[  5]   0.00-10.01  sec  51.5 MBytes  43.1 Mbits/sec                  receiver
[  7]   0.00-10.00  sec   103 MBytes  86.7 Mbits/sec    0             sender
[  7]   0.00-10.01  sec   102 MBytes  85.4 Mbits/sec                  receiver
[  9]   0.00-10.00  sec   141 MBytes   118 Mbits/sec    0             sender
[  9]   0.00-10.01  sec   140 MBytes   117 Mbits/sec                  receiver
[ 11]   0.00-10.00  sec   103 MBytes  86.5 Mbits/sec    0             sender
[ 11]   0.00-10.01  sec   102 MBytes  85.4 Mbits/sec                  receiver
[ 13]   0.00-10.00  sec   103 MBytes  86.5 Mbits/sec    0             sender
[ 13]   0.00-10.01  sec   102 MBytes  85.4 Mbits/sec                  receiver
[ 15]   0.00-10.00  sec   142 MBytes   119 Mbits/sec    0             sender
[ 15]   0.00-10.01  sec   141 MBytes   118 Mbits/sec                  receiver
[ 17]   0.00-10.00  sec   103 MBytes  86.1 Mbits/sec    0             sender
[ 17]   0.00-10.01  sec   102 MBytes  85.2 Mbits/sec                  receiver
[ 19]   0.00-10.00  sec   104 MBytes  87.1 Mbits/sec    0             sender
[ 19]   0.00-10.01  sec   103 MBytes  86.1 Mbits/sec                  receiver
[ 21]   0.00-10.00  sec   142 MBytes   119 Mbits/sec    0             sender
[ 21]   0.00-10.01  sec   141 MBytes   118 Mbits/sec                  receiver
[ 23]   0.00-10.00  sec   142 MBytes   119 Mbits/sec    0             sender
[ 23]   0.00-10.01  sec   141 MBytes   118 Mbits/sec                  receiver
[SUM]   0.00-10.00  sec  1.11 GBytes   951 Mbits/sec    0             sender
[SUM]   0.00-10.01  sec  1.10 GBytes   941 Mbits/sec                  receiver


# Run basic TCP test with 10 parallel streams in reverse
$ iperf3 -c 10.42.42.150 -P 10 -R
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   113 MBytes  95.1 Mbits/sec  334             sender
[  5]   0.00-10.00  sec   112 MBytes  94.2 Mbits/sec                  receiver
[  7]   0.00-10.00  sec   113 MBytes  95.0 Mbits/sec  284             sender
[  7]   0.00-10.00  sec   112 MBytes  94.2 Mbits/sec                  receiver
[  9]   0.00-10.00  sec   113 MBytes  95.1 Mbits/sec  304             sender
[  9]   0.00-10.00  sec   112 MBytes  94.0 Mbits/sec                  receiver
[ 11]   0.00-10.00  sec   113 MBytes  94.9 Mbits/sec  284             sender
[ 11]   0.00-10.00  sec   112 MBytes  94.2 Mbits/sec                  receiver
[ 13]   0.00-10.00  sec   113 MBytes  94.9 Mbits/sec  301             sender
[ 13]   0.00-10.00  sec   112 MBytes  94.0 Mbits/sec                  receiver
[ 15]   0.00-10.00  sec   113 MBytes  94.8 Mbits/sec  302             sender
[ 15]   0.00-10.00  sec   112 MBytes  94.0 Mbits/sec                  receiver
[ 17]   0.00-10.00  sec   113 MBytes  94.9 Mbits/sec  327             sender
[ 17]   0.00-10.00  sec   112 MBytes  94.0 Mbits/sec                  receiver
[ 19]   0.00-10.00  sec   113 MBytes  95.1 Mbits/sec  313             sender
[ 19]   0.00-10.00  sec   112 MBytes  94.0 Mbits/sec                  receiver
[ 21]   0.00-10.00  sec   113 MBytes  94.9 Mbits/sec  402             sender
[ 21]   0.00-10.00  sec   112 MBytes  94.0 Mbits/sec                  receiver
[ 23]   0.00-10.00  sec   113 MBytes  94.8 Mbits/sec  311             sender
[ 23]   0.00-10.00  sec   112 MBytes  94.0 Mbits/sec                  receiver
[SUM]   0.00-10.00  sec  1.11 GBytes   950 Mbits/sec  3162             sender
[SUM]   0.00-10.00  sec  1.10 GBytes   941 Mbits/sec                  receiver


```

### UDP Tests
```shell
# Run UDP test with 1000 Mbit/sec connection
$ iperf3 -c 10.42.42.150 -u -b 1000M
Connecting to host 10.42.42.150, port 5201
[  5] local 10.42.42.228 port 57600 connected to 10.42.42.150 port 5201
[ ID] Interval           Transfer     Bitrate         Total Datagrams
[  5]   0.00-1.00   sec   114 MBytes   957 Mbits/sec  82577  
[  5]   1.00-2.00   sec   114 MBytes   956 Mbits/sec  82532  
[  5]   2.00-3.00   sec   114 MBytes   957 Mbits/sec  82590  
[  5]   3.00-4.00   sec   114 MBytes   956 Mbits/sec  82562  
[  5]   4.00-5.00   sec   114 MBytes   956 Mbits/sec  82526  
[  5]   5.00-6.00   sec   114 MBytes   957 Mbits/sec  82590  
[  5]   6.00-7.00   sec   114 MBytes   956 Mbits/sec  82545  
[  5]   7.00-8.00   sec   114 MBytes   957 Mbits/sec  82587  
[  5]   8.00-9.00   sec   114 MBytes   956 Mbits/sec  82532  
[  5]   9.00-10.00  sec   114 MBytes   956 Mbits/sec  82561  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.00  sec  1.11 GBytes   956 Mbits/sec  0.000 ms  0/825602 (0%)  sender
[  5]   0.00-10.00  sec  1.11 GBytes   956 Mbits/sec  0.011 ms  0/825580 (0%)  receiver


# Run UDP test with 1000 Mbit/sec connection and with 10 parallel streams
$ iperf3 -c 10.42.42.150 -u -b 1000M -P 10
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.00  sec  95.2 MBytes  79.8 Mbits/sec  0.000 ms  0/68907 (0%)  sender
[  5]   0.00-10.01  sec  95.2 MBytes  79.8 Mbits/sec  0.038 ms  0/68907 (0%)  receiver
[  7]   0.00-10.00  sec  95.1 MBytes  79.8 Mbits/sec  0.000 ms  0/68879 (0%)  sender
[  7]   0.00-10.01  sec  95.1 MBytes  79.7 Mbits/sec  0.080 ms  0/68879 (0%)  receiver
[  9]   0.00-10.00  sec   143 MBytes   120 Mbits/sec  0.000 ms  0/103284 (0%)  sender
[  9]   0.00-10.01  sec   143 MBytes   120 Mbits/sec  0.210 ms  0/103284 (0%)  receiver
[ 11]   0.00-10.00  sec  95.4 MBytes  80.0 Mbits/sec  0.000 ms  0/69076 (0%)  sender
[ 11]   0.00-10.01  sec  95.4 MBytes  80.0 Mbits/sec  0.133 ms  0/69076 (0%)  receiver
[ 13]   0.00-10.00  sec  95.0 MBytes  79.7 Mbits/sec  0.000 ms  0/68806 (0%)  sender
[ 13]   0.00-10.01  sec  95.0 MBytes  79.6 Mbits/sec  0.227 ms  0/68806 (0%)  receiver
[ 15]   0.00-10.00  sec   143 MBytes   120 Mbits/sec  0.000 ms  0/103478 (0%)  sender
[ 15]   0.00-10.01  sec   143 MBytes   120 Mbits/sec  0.297 ms  0/103478 (0%)  receiver
[ 17]   0.00-10.00  sec   143 MBytes   120 Mbits/sec  0.000 ms  0/103268 (0%)  sender
[ 17]   0.00-10.01  sec   143 MBytes   120 Mbits/sec  0.122 ms  0/103268 (0%)  receiver
[ 19]   0.00-10.00  sec  94.9 MBytes  79.6 Mbits/sec  0.000 ms  0/68720 (0%)  sender
[ 19]   0.00-10.01  sec  94.9 MBytes  79.5 Mbits/sec  0.406 ms  0/68720 (0%)  receiver
[ 21]   0.00-10.00  sec  95.0 MBytes  79.7 Mbits/sec  0.000 ms  0/68790 (0%)  sender
[ 21]   0.00-10.01  sec  95.0 MBytes  79.6 Mbits/sec  0.048 ms  0/68789 (0%)  receiver
[ 23]   0.00-10.00  sec   142 MBytes   119 Mbits/sec  0.000 ms  0/102976 (0%)  sender
[ 23]   0.00-10.01  sec   142 MBytes   119 Mbits/sec  0.041 ms  0/102976 (0%)  receiver
[SUM]   0.00-10.00  sec  1.11 GBytes   957 Mbits/sec  0.000 ms  0/826184 (0%)  sender
[SUM]   0.00-10.01  sec  1.11 GBytes   956 Mbits/sec  0.160 ms  0/826183 (0%)  receiver


# Run UDP test with 1000 Mbit/sec connection and with 10 parallel streams in reverse
$ iperf3 -c 10.42.42.150 -u -b 1000M -P 10 -R
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.00  sec   118 MBytes  98.6 Mbits/sec  0.000 ms  0/0 (0%)  sender
[  5]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.060 ms  2534/85106 (3%)  receiver
[  7]   0.00-10.00  sec   118 MBytes  98.6 Mbits/sec  0.000 ms  0/0 (0%)  sender
[  7]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.065 ms  2534/85106 (3%)  receiver
[  9]   0.00-10.00  sec   118 MBytes  98.6 Mbits/sec  0.000 ms  0/0 (0%)  sender
[  9]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.086 ms  2534/85106 (3%)  receiver
[ 11]   0.00-10.00  sec   118 MBytes  98.6 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 11]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.068 ms  2534/85106 (3%)  receiver
[ 13]   0.00-10.00  sec   118 MBytes  98.6 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 13]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.069 ms  2534/85105 (3%)  receiver
[ 15]   0.00-10.00  sec   118 MBytes  98.6 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 15]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.072 ms  2534/85105 (3%)  receiver
[ 17]   0.00-10.00  sec   118 MBytes  98.6 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 17]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.071 ms  2534/85105 (3%)  receiver
[ 19]   0.00-10.00  sec   118 MBytes  98.6 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 19]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.070 ms  2534/85105 (3%)  receiver
[ 21]   0.00-10.00  sec   118 MBytes  98.6 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 21]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.068 ms  2534/85105 (3%)  receiver
[ 23]   0.00-10.00  sec   118 MBytes  98.6 Mbits/sec  0.000 ms  0/0 (0%)  sender
[ 23]   0.00-10.00  sec   114 MBytes  95.6 Mbits/sec  0.068 ms  2534/85104 (3%)  receiver
[SUM]   0.00-10.00  sec  1.15 GBytes   986 Mbits/sec  0.000 ms  0/0 (0%)  sender
[SUM]   0.00-10.00  sec  1.11 GBytes   956 Mbits/sec  0.070 ms  25340/851053 (3%)  receiver
```

## Conclusion
TrueNAS SCALE's core features were straightforward to use, although I primarily focused on ZFS and sharing capabilities, so it's difficult to assess the ease of managing containers and virtual machines. The primary reason I didn't explore those features further was because soon after installation I discovered the operating system wouldn't allow me to utilize the NVMe drive where the OS was installed for anything else. The prospect of sacrificing the entire 500GB NVMe solely for the OS wasn't appealing.

From a security perspective, the system seems somewhat open out of the box. The web UI access isn't HTTPS by default, and the local terminal is accessible via a USB keyboard and monitor. This places the burden of security hardening on the user, potentially requiring those with less technical experience to study documentation and tutorials.

Overall, TrueNAS SCALE appears to be a solid product, and it's a shame that the OS takes the entire disk. Otherwise I would have invested more time exploring its other features.

## TrueNAS Videos
Lawrence Systems on YouTube is an excellent resource for learning more about TrueNAS.

[Link to the channel](https://www.youtube.com/@LAWRENCESYSTEMS)
