---
title: Terramaster F4-424 Pro UEFI
author: tanezky
date: 2024-08-09 20:00:00 +/-TTTT
description: Time to take a look inside the UEFI and make some power measurements.
categories: [NAS]
tags: [NAS,Terramaster,F4-424 Pro, UEFI]
image: /assets/img/posts/terramaster-uefi/terramaster-uefi-post.jpg
---
![PC health tab](/assets/img/posts/terramaster-uefi/uefi01_1.jpg){: width="1920" height="1080" .w-75 .center}
_UEFI Main tab_
The firmware is based on AMI (American Megatrends International) Aptio, and the version for the F4-424 is MADN0101.V04. It offers a wide array of configuration options, but for my purposes, the most interesting features are support for fTPM (Firmware Trusted Platform Module) and the ability to roll in my own secure boot keys. While UEFI itself isn't a bootloader, this version can be configured to execute specific UEFI applications (Enroll Efi Image), potentially allowing the use of UKI (Unified Kernel Image) and bypassing the need for a separate bootloader.

I used Figma to capture screenshots of the entire UEFI interface (press the play button in the right upper corner for mouse navigation between screens).
[Figma page](https://www.figma.com/design/1m9lEXdQhvGAz20QHBVGpO/F4-424-Pro-EFI%2FBIOS?node-id=1-75&t=zP4OR0KNAdO7bxBU-1)

## Power Measurements and Fan Utilization
For power measurements, I used a [Refoss P11](https://refoss.net/products/refoss-tesmota-wi-fi-plug-p11) Wifi smart plug with integrated power metering. These plugs, part of my smart home setup, come with Tasmota pre-installed, allowing me to run them locally without cloud connections, which is a core requirement for my home automation.

While the P11 is a consumer-grade product and its power measurements might not be highly precise, it provides a reasonable estimate, which is sufficient for my needs. I compared its readings with an older, cheap SilverCrest PM334-37853 energy monitor, and the values were very close, further validating the P11's accuracy for my purposes.

![PC health tab](/assets/img/posts/terramaster-uefi/uefi01.jpg){: width="1920" height="1080" .w-75 .center}
_Fan utilisation on maximum_
![PWM Max setting](/assets/img/posts/terramaster-uefi/uefi02.jpg){: width="1920" height="1080" .w-75 .center}
_100% PWM setting_
The UEFI's "PC Health" tab displays temperature and fan utilization information. Fan utilization thresholds can be configured based on CPU temperature. For testing, I set the mode to manual and maxed out the value at 255 (100% utilization). Surprisingly, the reported fan speed in the UEFI was around 1566 RPM (±5 RPM), which is about half the fan's [datasheet](/assets/files/datasheets/snowfan-1-2106260UQ9326.pdf) stated maximum of 3000 RPM. This suggests the fan could potentially be replaced with a quieter PC chassis fan running at a similar speed.

Regarding power usage, I first checked the system without any hard drives attached, only the Samsung NVMe SSD and USB connected. At startup, it consumed around 36W, dropping to 15W at idle. With the fan at 100% utilization, power consumption increased to 16W.

Connecting all four HDDs significantly lengthened the boot time, with power consumption reaching 48W during boot and settling at 35W after a while with a ~950 RPM fan speed. At 100% fan utilization, power consumption rose to 43W.

Connecting my USB-C dock (used for additional USB ports) increased consumption by about 3 watts.
CPU temperatures were between 47°C and 55°C, depending on the fan speed and system temperatures between 37°C and 39°C

I'm slightly skeptical about these measurements, as the UEFI seems to be interacting with the blank disks even though all disk tests are disabled in the UEFI settings. However, as ballpark figures the NAS consumes roughly 16W at idle in the UEFI without disks and around 35W with disks.

## Secure Boot Vulnerability
![Compromised PK key](/assets/img/posts/terramaster-uefi/uefi03.jpg){: width="1920" height="1080" .w-75 .center}
_Compromised PK key_
A recent discovery by Binarly revealed a widespread [vulnerability](https://arstechnica.com/security/2024/07/secure-boot-is-completely-compromised-on-200-models-from-5-big-device-makers/) affecting numerous manufacturers, including TerraMaster. They did not replace the default AMI test platform key on their devices, allowing malicious actors to potentially bypass Secure Boot entirely.

Upon inspecting the platform key (PK) certificate on my F4-424 Pro, I found that the PK certificate legend matches the GUID of the leaked keys. This confirms the vulnerability's presence.

Therefore, it's strongly advised against using the default Secure Boot configuration on this device. To mitigate this risk, I'll need to generate and enroll new keys during the final setup process.

