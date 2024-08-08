---
title: Inspecting Terramaster F4-424 Pro
author: tanezky
date: 2024-07-24 21:00:00 +/-TTTT
description: Closer look inside the Terramaster F4-424 Pro NAS device
categories: [NAS]
tags: [NAS,Terramaster,F4-424 Pro]
image: /assets/img/posts/disassembly-of-terramaster-f4-424-pro/inv_post.jpg
---

## The Teardown Ritual

Throughout my career, I've made a habit of performing quick hardware reviews for potential third-party off-the-shelf devices. I also apply this practice at home to anticipate potential issues that might arise in the future. The TerraMaster F4-424 Pro was no exception, given its crucial role in my homelab and the vital services it will host for my home network. 

Fortunately, there were no pesky warranty stickers to worry about, just plastic, metal, and circuit boards ready to be explored.


## First Look
![Motherboard connector](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/disassembly01.jpg){: width="3852" height="1846" .w-75 .center}
_USB thumbdrive and drive bay connector_

A few interesting design choices became apparent as I started poking around. As expected for a NAS, the operating system or bootloader lives on a USB thumb drive plugged directly into the motherboard.

Less expected was the connection for the drive bay expansion card: a card-edge connector. When I removed the motherboard, this connection felt a bit loose. Given that the NAS is likely to experience some expansion and contraction due to heat cycles, I'm a little concerned about the long-term reliability of this connection. Only time will tell if it'll hold up or need some occasional tightening.

_Note:_ I temporarily removed one of the support legs to get a better shot of the USB drive and card-edge connection for the photo.

![Fan connector](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/disassembly02.jpg){: width="2813" height="1377" .w-75 .center}
_On the otherside there is a 4 pin fan connector_


## Complete Disassembly
![All parts](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/disassembly03.jpg){: width="2383" height="1848" .w-75 .center}
_Disassembled device_


## The Fan
![Fan from front](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/disassembly04.jpg){: width="2725" height="1848" .w-75 .center}
_Frontside of the fan_

![Fan from back](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/disassembly05.jpg){: width="2602" height="1848" .w-75 .center}
_Backside of the fan_

The TerraMaster F4-424 Pro is cooled by a 120 x 120 x 25 mm seven-blade fan manufactured by Shenzhen Yongyihao Electronics Co. Ltd., better known for their SnowFAN brand. The fan operates between 8-13.5 VDC, consumes 3.84 Watts, and can reach a maximum speed of 3000 RPM, according to the [datasheet](/assets/files/datasheets/snowfan-1-2106260UQ9326.pdf). The manufacturer [claims](https://www.snowfan.hk/dc_axial_fan/107.html) a lifespan of up to 70,000 hours, potentially translating to eight years of continuous operation.

While these specifications are promising, real-world performance and longevity under continuous load remain to be seen. Fortunately, [Techpowerup](https://www.techpowerup.com/review/terramaster-d8-hybrid/9.html) has conducted a review with noise level measurements on this same fan model, which can give some insights into its expected performance.


## SATA Expansion Board
![Expansion board from front](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/disassembly06.jpg){: width="3861" height="1774" .w-75 .center}
_Frontside of the expansion board_

![Expansion board from back](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/disassembly07.jpg){: width="3544" height="1660" .w-75 .center}
_Backside of the expansion board_

The SATA expansion board offers four drive slots, but a closer look at the PCB traces reveals a noteworthy configuration. SATA1 and SATA2 connect directly to the motherboard, while SATA3 and SATA4 are routed through an [Asmedia ASM1061](https://www.asmedia.com.tw/product/77BYq58SX3HyepH7/58dYQ8bxZ4UR9wG5) PCIe to SATA controller.
![Closer view of ASM 1061 chip](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/disassembly08.jpg){: width="1848" height="1703" .w-25 .right}

According to the [datasheet](/assets/files/datasheets/ASM1061_Data%20Sheet_R1_8.pdf), this controller supports PCIe 2.0, which, while adequate for most SATA SSDs, could theoretically bottleneck the bandwidth available to the SATA3 and SATA4 drives due to the older PCIe standard. The controller offers two Serial ATA PHYs for 1.5, 3.0, and 6 GHz signaling, theoretically supporting SATA revisions 1.5, 2.0, and 3.0, along with the AHCI Specification.

This design raises questions about theoretical performance limitations for drives connected to SATA3 and SATA4 due to the older PCIe 2.0 interface and the use of a third-party controller. It's also possible that this configuration could limit or create stability issues with hardware passthrough to virtual resources. Further testing will be necessary to determine the actual impact on drive performance and compatibility with virtualization features.


## RAM Module
![RAM module from front](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/disassembly09.jpg){: width="3311" height="1660" .w-75 .center}
_Frontside of the RAM module_
![RAM module from back](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/disassembly10.jpg){: width="3594" height="1732" .w-75 .center}
_Backside of the RAM module_

Interestingly, the RAM module has TerraMaster branding on its chips, but details about its specifications are elusive online. While a replacement module (A-SRAMD5-32G) is available on their [website](https://www.terra-master.com/global/products/a-sramd5-32g.html?page=desc), the price tag of around 200€ is rather steep. This has raised concerns among some users, especially as many have reported compatibility [issues](https://forum.terra-master.com/en/viewtopic.php?t=2283&start=60) with third-party RAM when running TerraMaster's TOS operating system.

On the bright side, user experiences on forums suggest that compatibility issues don't seem to arise when using alternative operating systems. Additionally, the device comes with a 2-year warranty, offering some reassurance that the RAM module should be reliable during that period. Nevertheless, the potential lock-in to TerraMaster's proprietary modules and their high cost remain a drawback for those using TOS.


## Motherboard

### Top
![Motherboard top view](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top01.jpg){: width="2923" height="1829" .w-75 .center}
_Motherboard top view_
![Motherboard with heatsink removed](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top02.jpg){: width="2488" height="1848" .w-75 .center}
_Motherboard with heatsink removed_
![Motherboard without heatsink](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top03.jpg){: width="3075" height="1848" .w-75 .center}
_Motherboard without heatsink_

![ICs 1](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top04.jpg){: width="1848" height="4000" .w-25 .normal}
![ICs 2](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top05.jpg){: width="1848" height="4000" .w-25 .normal}
![ICs 3](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top07.jpg){: width="4000" height="1848" .w-25 .normal}
![ICs 4](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top08.jpg){: width="4000" height="1848" .w-25 .normal}
![ICs 5](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top09.jpg){: width="4000" height="1848" .w-25 .normal}
![ICs 6](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top10.jpg){: width="4000" height="1848" .w-25 .normal}
![ICs 7](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top11.jpg){: width="4000" height="1848" .w-25 .normal}
![ICs 8](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top12.jpg){: width="4000" height="1848" .w-25 .normal}
![ICs 9](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top13.jpg){: width="4000" height="1848" .w-25 .normal}
![ICs 10](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top14.jpg){: width="4000" height="1848" .w-25 .normal}

![Ethernet controllers](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-top06.jpg){: width="4000" height="1848" .w-75 .center}
_Ethernet controllers_

The motherboard, mounted upside down to expose the NVMe slots through the side panel, features a large heatsink, terminals, a buzzer, a CMOS battery, and jumpers for resetting the CMOS. Notably, accessing the CMOS battery and jumpers requires removing the motherboard, which could be inconvenient for maintenance or troubleshooting.

A silica thermal pad facilitates thermal conductivity between the CPU and heatsink. While this is a common solution, its effectiveness under heavy CPU loads remains to be seen and will require further testing to ensure the CPU operates within safe temperature limits. Since these pads are typically single-use and the heatsink was removed during inspection, the pad's performance may be compromised.  It's recommended to replace it with high-quality thermal paste for optimal cooling.

The CMOS battery is a standard CR2032 3V Lithium Manganese Dioxide (Li-MnO2) coin battery manufactured by EVE. Its primary function is to maintain the real-time clock (RTC) and keep the time accurate. According to the [datasheet](/assets/files/datasheets/eve-cr2032.pdf) its maximum operating temperature is +70°C and a nominal capacity of 225mAh, the battery's expected lifespan varies depending on usage patterns, ambient temperature, and individual battery quality. While it's typically estimated to last around five years when the device is powered on and 2.5-3 years when unpowered, its real-world longevity in this specific NAS setup remains to be seen.

The ethernet controllers are Realtek [RTL8125BG](https://www.realtek.com/Product/Index?id=3962) 10/100/1000M/2.5G chips connected to PCIe. Linux Kernel support for these controllers was introduced in version [5.9](https://linuxreviews.org/Realtek_RTL_8125) and is available on mainstream Linux distributions like [Ubuntu 22.04](https://en.wikipedia.org/wiki/Ubuntu_version_history#Table_of_versions) and [Debian 11](https://en.wikipedia.org/wiki/Debian_version_history) (Bullseye).

Even though the device has two 2.5G Ethernet controllers, there's no mention of support for 802.3ad (LACP) or 802.1AX (Link Aggregation) in the [datasheet](/assets/files/datasheets/RTL8125BG-datasheet.pdf). This could be a potential limitation for future when seeking to maximize network throughput 


### Bottom
![Motherboard bottom view](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-bot01.jpg){: width="3056" height="1847" .w-75 .center}
_Motherboard bottom view_

![ICs 1](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-bot02.jpg){: width="1848" height="1675" .w-25 .normal}
![ICs 1](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-bot03.jpg){: width="1848" height="1561" .w-25 .normal}
![ICs 1](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-bot04.jpg){: width="1848" height="1703" .w-25 .normal}
![ICs 1](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-bot05.jpg){: width="2332" height="1847" .w-25 .normal}
![ICs 1](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-bot06.jpg){: width="2858" height="1848" .w-25 .normal}
![ICs 1](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-bot07.jpg){: width="2007" height="1848" .w-25 .normal}
![ICs 1](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-bot08.jpg){: width="1799" height="1848" .w-25 .normal}
![ICs 1](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-bot09.jpg){: width="2699" height="1846" .w-25 .normal}
![ICs 1](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-bot10.jpg){: width="1818" height="1848" .w-25 .normal}

![Motherboard LEDs](/assets/img/posts/disassembly-of-terramaster-f4-424-pro/mb-bot11.jpg){: width="4000" height="1848" .w-75 .center}
_Motherboard LEDs_

The bottom side of the motherboard, which is actually facing upwards within the device, houses the system and HDD LEDs, a single RAM slot, two M.2 SSD 2280 slots, and a reset button. The Winbond 25Q128JVSQ chip labeled U2 is likely the CMOS chip, responsible for storing BIOS settings and other system configurations.

## Conclusion

A close examination of the PCB with a magnifying lamp revealed no soldering issues and generally good build quality. The absence of any unusual odor is also a positive indicator, as strong smells from PCBs can sometimes signal poor or outdated manufacturing practices with inadequate ventilation.

However, the SATA expansion board, using an ASM1061 controller, raises questions about potential performance limitations when using multiple hard drives concurrently. While the ASM1061 is still a capable controller on paper, its PCIe 2.0 interface might become a bottleneck for demanding workloads. Additionally, the use of a third-party controller could introduce compatibility challenges with certain kernels or virtualization setups.

The thermal pad needs to be replaced with thermal paste, while necessary for optimal cooling, presents a trade-off. Although high-quality thermal paste offers lower thermal resistance and improved cooling efficiency, it tends to dry out over time and will require replacement within 3-5 years.

Furthermore, the lack of explicit support for IEEE 802.3ad (LACP) and 802.1AX (Link Aggregation) in the Ethernet controllers' documentation could limit future network upgrades beyond 2.5G speeds.

Overall, the device appears well-made at first glance, but these potential issues require further investigation and testing to determine their real-world impact on performance and functionality.
