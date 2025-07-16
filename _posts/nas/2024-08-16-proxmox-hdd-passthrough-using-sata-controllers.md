---
title: Proxmox - HDD Passthrough Using SATA Controllers
author: tanezky
date: 2024-08-16 20:00:00 +/-TTTT
description: Testing ASM1061 passthrough on Terramaster F4-424 Pro
categories: [NAS Project, Reviewing Terramaster F4-424 Pro]
tags: [NAS,Terramaster F4-424 Pro, Proxmox, TrueNAS]
image: /assets/img/posts/pve-asm-pt/pve-asm-pt-post.jpg
---

> **2025 Update:** Decided to revisit this with new tricks.
{: .prompt-info }




### Diagnosing Issues

> tl;dr ASM1061 did not work with hardware passthrough

After trying various passthrough techniques in Proxmox, the ASMedia ASM1061 controller refused to work in a virtual machine. While the passthrough was configured correctly on the host, the guest VM would fail to initialize the chip, presenting an `unknown header type 7f` error. Further investigation was done which uncovered hardware and firmware faults in both the chip and the host system.

#### Motherboard BIOS Errors

The dmesg output was filled with messages like this:

```shell
Error (bug): Could not resolve symbol [\_SB.PC00.TXHC.RHUB.SS01], AE_NOT_FOUND
Error (bug): Could not resolve symbol [\_SB.UBTC.RUCC], AE_NOT_FOUND
```

These ACPI (Advanced Configuration and Power Interface) errors indicate that the motherboard's BIOS/UEFI is providing a faulty or incomplete description of the hardware to the operating system. Since reliable power management and system-wide stability are critical for virtualization, a buggy BIOS can create an unstable foundation for passthrough. The only way to fix these errors is for Terramaster to provide a BIOS update with the necessary ACPI fixes..

BIOS version

> `DMI: retsamarret 000-F4424Pro-CN36-2000/M-ADLN01, BIOS MADN0101.V04 12/22/2023`

**Note** BIOS errors were already present when trying out Terramaster's [TOS 5.1.145](/posts/terramaster-os/#inspecting-dmesg).

#### Function-Level Reset (FLR)

For a stable PCI passthrough the hypervisor (Proxmox/VFIO) must be able to reset a device before handing it to a virtual machine. This ensures the hardware is in a clean state every time the VM starts or reboots.

The most reliable and preferred method is Function-Level Reset (FLR). An `lspci -vvv` command on the host revealed the chip's limitation:

```shell
Capabilities: [80] Express (v2) Legacy Endpoint, MSI 00
  DevCap: MaxPayload 512 bytes, PhantFunc 0, Latency L0s <1us, L1 <8us
  ExtTag- AttnBtn- AttnInd- PwrInd- RBE+ FLReset-
```

The `FLReset-` flag confirms this chip does not support the modern standard for PCI resets, which is a major red flag for passthrough.

#### Investigating Other Reset Mechanisms

Without FLR the system relies on traditional methods like Power Management (pm) and Bus (bus) resets. While the host reported these methods were available, experiments showed the chip couldn't handle them correctly.

Further investigation into the chip's power management revealed more signs of instability. When its driver was unbound on the host, the chip occassionally reported its power state as `unknown` instead of a standard state like D0 (On) or D3 (Sleep).
```shell
# cat /sys/bus/pci/devices/0000:03:00.0/power_state
unknown
```

This indicates the chip's firmware doesn't correctly communicate with the kernel's power management system.

#### Conclusion

The investigation leads to a clear verdict: the ASMedia ASM1061 controller on Terramaster F4-424 Pro is fundamentally incompatible with VFIO passthrough due to hardware and firmware limitations.

The failure is caused by a combinations of two factors

1. The motherboard's BIOS contains numerous ACPI errors, this creates an unstable foundation for virtualization.
2. The ASM1061 chip itself does not support FLR and has a faulty power management implementation, making it unable to handle the mandatory resets required by the hypervisor.

If disk passthrough is required, the integrated Intel SATA controller (00:17.0) for HDD bays 1 and 2 is viable option. The ASMedia controlled bays 3 and 4 should not be used for this purpose.

## Environment Preparation

### Installing TrueNAS

#### Getting ISO
![Get TrueNAS ISO](/assets/img/posts/pve-asm-pt/pve-asm-pt01.jpg){: width="1307" height="574" .center}

1. local storage
2. ISO Images
3. Download from URL
4. Get latest url from Truenas [download page](https://www.truenas.com/download-truenas-scale/)
5. Query URL to get `File name` populated
6. Hash algorithm: SHA-256, copy paste checksum from Truenas download page
7. Download

#### Create Virtual Machine

If not mentioned, use default value.

- OS
  - ISO image: Select TruenasScale ISO
- System
  - Machine: q35
  - BIOS: OVMF(UEFI)
  - Add EFI Disk
  - EFI Storage: local-lvm
- CPU
  - Cores: 4
  - Type: x86-64-v2-AES
- Memory
  - Memory (MiB): 16384

#### Starting Installation
![Start Installation](/assets/img/posts/pve-asm-pt/pve-asm-pt02.jpg){: width="1307" height="569" .center}

Starting the installation was a bit tricky.

1. Select VM
2. Select Console and start VM
3. Expand toolbar from left, the key button comes available when VM starts
4. Hit CTRL+DEL button to restart VM
5. Hit Esc button when VM starts to enter UEFI settings, repeat from step 4 if UEFI settings does not come up.

#### UEFI Settings
![Disable Secure Boot](/assets/img/posts/pve-asm-pt/pve-asm-pt03.jpg){: width="889" height="497" .center}
_Disable Secure Boot_
`Device Manager --> Secure Boot Configuration --> Untick "Attempt Secure Boot"`

Return to `Main Page` with `Esc` and select `Reset`

The installation should start.

### Assign SATA Controller

Find out addresses for SATA controllers (00:17.0 and 03:00.0).
On Terramaster F4-424 Pro there are two controllers, Intel (HDD bays 1,2) and ASM1061 (HDD bays 3,4).
```shell
lspci -knn
00:00.0 Host bridge [0600]: Intel Corporation Device [8086:4617]
	DeviceName: Onboard - Other
	Subsystem: Intel Corporation Device [8086:7270]
	Kernel driver in use: igen6_edac
	Kernel modules: igen6_edac
00:02.0 VGA compatible controller [0300]: Intel Corporation Alder Lake-N [UHD Graphics] [8086:46d0]
	DeviceName: Onboard - Video
	Subsystem: Intel Corporation Alder Lake-N [UHD Graphics] [8086:7270]
	Kernel driver in use: i915
	Kernel modules: i915, xe
00:14.0 USB controller [0c03]: Intel Corporation Alder Lake-N PCH USB 3.2 xHCI Host Controller [8086:54ed]
	DeviceName: Onboard - Other
	Subsystem: Intel Corporation Alder Lake-N PCH USB 3.2 xHCI Host Controller [8086:7270]
	Kernel driver in use: xhci_hcd
	Kernel modules: xhci_pci
00:14.2 RAM memory [0500]: Intel Corporation Alder Lake-N PCH Shared SRAM [8086:54ef]
	DeviceName: Onboard - Other
	Subsystem: Intel Corporation Alder Lake-N PCH Shared SRAM [8086:7270]
00:16.0 Communication controller [0780]: Intel Corporation Alder Lake-N PCH HECI Controller [8086:54e0]
	DeviceName: Onboard - Other
	Subsystem: Intel Corporation Alder Lake-N PCH HECI Controller [8086:7270]
	Kernel driver in use: mei_me
	Kernel modules: mei_me
00:17.0 SATA controller [0106]: Intel Corporation Alder Lake-N SATA AHCI Controller [8086:54d3]
	DeviceName: Onboard - SATA
	Subsystem: Intel Corporation Alder Lake-N SATA AHCI Controller [8086:7270]
	Kernel driver in use: ahci
	Kernel modules: ahci
00:1c.0 PCI bridge [0604]: Intel Corporation Alder Lake-N PCI Express Root Port [8086:54ba]
	Subsystem: Intel Corporation Alder Lake-N PCI Express Root Port [8086:7270]
	Kernel driver in use: pcieport
00:1c.3 PCI bridge [0604]: Intel Corporation Device [8086:54bb]
	Subsystem: Intel Corporation Device [8086:7270]
	Kernel driver in use: pcieport
00:1c.6 PCI bridge [0604]: Intel Corporation Alder Lake-N PCI Express Root Port [8086:54be]
	Subsystem: Intel Corporation Alder Lake-N PCI Express Root Port [8086:7270]
	Kernel driver in use: pcieport
00:1d.0 PCI bridge [0604]: Intel Corporation Alder Lake-N PCI Express Root Port [8086:54b0]
	Subsystem: Intel Corporation Alder Lake-N PCI Express Root Port [8086:7270]
	Kernel driver in use: pcieport
00:1f.0 ISA bridge [0601]: Intel Corporation Alder Lake-N PCH eSPI Controller [8086:5481]
	DeviceName: Onboard - Other
	Subsystem: Intel Corporation Alder Lake-N PCH eSPI Controller [8086:7270]
00:1f.3 Audio device [0403]: Intel Corporation Alder Lake-N PCH High Definition Audio Controller [8086:54c8]
	DeviceName: Onboard - Sound
	Subsystem: Intel Corporation Alder Lake-N PCH High Definition Audio Controller [8086:7270]
	Kernel driver in use: snd_hda_intel
	Kernel modules: snd_hda_intel, snd_sof_pci_intel_tgl
00:1f.4 SMBus [0c05]: Intel Corporation Alder Lake-N SMBus [8086:54a3]
	DeviceName: Onboard - Other
	Subsystem: Intel Corporation Alder Lake-N SMBus [8086:7270]
	Kernel driver in use: i801_smbus
	Kernel modules: i2c_i801
00:1f.5 Serial bus controller [0c80]: Intel Corporation Alder Lake-N SPI (flash) Controller [8086:54a4]
	DeviceName: Onboard - Other
	Subsystem: Intel Corporation Alder Lake-N SPI (flash) Controller [8086:7270]
	Kernel driver in use: intel-spi
	Kernel modules: spi_intel_pci
01:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller [10ec:8125] (rev 05)
	Subsystem: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller [10ec:0123]
	Kernel driver in use: r8169
	Kernel modules: r8169
02:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller [10ec:8125] (rev 05)
	Subsystem: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller [10ec:0123]
	Kernel driver in use: r8169
	Kernel modules: r8169
03:00.0 SATA controller [0106]: ASMedia Technology Inc. ASM1062 Serial ATA Controller [1b21:0612] (rev 02)
	Subsystem: ASMedia Technology Inc. ASM1061/ASM1062 Serial ATA Controller [1b21:1060]
	Kernel driver in use: ahci
	Kernel modules: ahci
04:00.0 Non-Volatile memory controller [0108]: Samsung Electronics Co Ltd NVMe SSD Controller 980 [144d:a809]
	Subsystem: Samsung Electronics Co Ltd NVMe SSD Controller 980 (DRAM-less) [144d:a801]
	Kernel driver in use: nvme
	Kernel modules: nvme

```

#### Add PCI Device to VM
![Add PCI Device](/assets/img/posts/pve-asm-pt/pve-asm-pt04.jpg){: width="1676" height="633" .center}
_Adding Intel SATA Controller to VM_

After confirming TrueNAS boots and is accessible through web-ui, power it down and add the PCI Device to the VM.

1. Select VM
2. Select Hardware
3. Add PCI Device
4. Raw Device: Choose address of a SATA controller
5. Tick PCI-Express

For mitigating issues it is best to first add one SATA controller, test it works with TrueNAS and then add second SATA controller.

### Verify PCI Device
![Verify Disks](/assets/img/posts/pve-asm-pt/pve-asm-pt05.jpg){: width="1475" height="715" .center}
Verify disks are visible in TrueNAS under `Storage --> Disks`.

Poweroff the VM and move on to add and test next PCI device.

## Tested Techniques

I tested following methods without connecting HDDs, since every time the ASM1061 crashed, the system required reboot to get the controller back into working state. The boot process on Terramaster F4-424 Pro takes considerably longer when disks are connected.

BIOS version
`DMI: retsamarret 000-F4424Pro-CN36-2000/M-ADLN01, BIOS MADN0101.V04 12/22/2023`

### Load ASM1061 with vfio-pci Kernel driver during boot

I received a [hint](https://github.com/tanezky/tanezky.github.io/issues/1) on possible [solution](https://gist.github.com/kiler129/4f765e8fdc41e1709f1f34f7f8f41706), the idea is to init ASM106x with vfio-pci before ahci.

**Steps**

```shell
# 1. Add vfio-pci to initramfs modules
echo "vfio-pci" >> /etc/initramfs-tools/modules

# Create modprobe script for initramfs (/usr/share/initramfs-tools/scripts/init-top/load_vfio-pci)
#!/bin/sh
modprobe vfio-pci ids=1b21:0612

# Make it executable
chmod +x /usr/share/initramfs-tools/scripts/init-top/load_vfio-pci

# Edit PREREQS="" in /usr/share/initramfs-tools/scripts/init-top/udev
PREREQS="load_vfio-pci"

# Update initrd and reboot
update-initramfs -d -c -k $(uname -r) && reboot
```
After reboot the ASM1061 was using vfio-pci driver. However, this didn't resolve the issue.

### Inspect ASM1061 ROM with rom-parser

I followed instruction on Proxmox PCI Passthrough [page](https://pve.proxmox.com/wiki/PCI_Passthrough#How_to_know_if_a_graphics_card_is_UEFI_.28OVMF.29_compatible) to extract ROM.

```shell
./rom-parser asm1061.rom 

Valid ROM signature found @0h, PCIR offset 40h
	PCIR: type 0 (x86 PC-AT), vendor: 1b21, device: 0612, class: 010400
	PCIR: revision 3, vendor revision: 420
	Last image
```
Based on the [page](https://pve.proxmox.com/wiki/PCI_Passthrough#How_to_know_if_a_graphics_card_is_UEFI_.28OVMF.29_compatible), the PCIR type should be **type 3** to be UEFI (ovmf) compatible.

#### ROM Version

The ASM1061 ROM version is 4.20

```shell
hexdump -n 512 -C asm1061.rom 

00000000  55 aa 4a e9 41 05 00 00  00 00 00 00 00 00 00 00  |U.J.A...........|
00000010  00 00 00 00 00 00 00 00  40 00 20 00 00 00 00 00  |........@. .....|
00000020  24 50 6e 50 01 02 1c 01  00 00 00 00 00 00 ad 00  |$PnP............|
00000030  c8 00 01 04 00 e4 14 04  54 04 f0 04 00 00 00 00  |........T.......|
00000040  50 43 49 52 21 1b 12 06  1c 00 1c 00 03 00 04 01  |PCIR!...........|
00000050  4a 00 20 04 00 80 41 00  00 00 00 00 11 06 12 06  |J. ...A.........|
00000060  01 06 02 06 14 06 15 06  00 00 00 00 00 00 00 00  |................|
00000070  43 6f 70 79 72 69 67 68  74 20 28 43 29 20 41 73  |Copyright (C) As|
00000080  6d 65 64 69 61 20 54 65  63 68 6e 6f 6c 6f 67 69  |media Technologi|
00000090  65 73 2c 20 49 6e 63 2e  20 41 6c 6c 20 52 69 67  |es, Inc. All Rig|
000000a0  68 74 20 72 65 73 65 72  76 65 64 2e 00 41 73 6d  |ht reserved..Asm|
000000b0  65 64 69 61 20 54 65 63  68 6e 6f 6c 6f 67 69 65  |edia Technologie|
000000c0  73 2c 20 49 6e 63 2e 00  41 73 6d 65 64 69 61 20  |s, Inc..Asmedia |
000000d0  31 30 36 58 20 53 41 54  41 2f 50 41 54 41 20 43  |106X SATA/PATA C|
000000e0  6f 6e 74 72 6f 6c 6c 65  72 20 56 65 72 20 34 2e  |ontroller Ver 4.|
000000f0  32 30 00 41 73 6d 65 64  69 61 20 31 30 36 58 20  |20.Asmedia 106X |
00000100  53 41 54 41 20 43 6f 6e  74 72 6f 6c 6c 65 72 20  |SATA Controller |
00000110  56 65 72 20 34 2e 32 30  00 89 c0 fc 24 50 6e 50  |Ver 4.20....$PnP|
00000120  01 02 3c 01 00 00 00 00  00 00 ad 00 c8 00 01 04  |..<.............|
00000130  00 e4 7c 03 54 04 58 04  00 00 00 00 24 50 6e 50  |..|.T.X.....$PnP|
00000140  01 02 5c 01 00 00 00 00  00 00 ad 00 c8 00 01 04  |..\.............|
00000150  00 e4 84 03 54 04 60 04  00 00 00 00 24 50 6e 50  |....T.`.....$PnP|
00000160  01 02 7c 01 00 00 00 00  00 00 ad 00 c8 00 01 04  |..|.............|
00000170  00 e4 8c 03 54 04 68 04  00 00 00 00 24 50 6e 50  |....T.h.....$PnP|
00000180  01 02 9c 01 00 00 00 00  00 00 ad 00 c8 00 01 04  |................|
00000190  00 e4 94 03 54 04 70 04  00 00 00 00 24 50 6e 50  |....T.p.....$PnP|
000001a0  01 02 bc 01 00 00 00 00  00 00 ad 00 c8 00 01 04  |................|
000001b0  00 e4 9c 03 54 04 78 04  00 00 00 00 24 50 6e 50  |....T.x.....$PnP|
000001c0  01 02 dc 01 00 00 00 00  00 00 ad 00 c8 00 01 04  |................|
000001d0  00 e4 a4 03 54 04 80 04  00 00 00 00 24 50 6e 50  |....T.......$PnP|
000001e0  01 02 fc 01 00 00 00 00  00 00 ad 00 c8 00 01 04  |................|
000001f0  00 e4 ac 03 54 04 88 04  00 00 00 00 24 50 6e 50  |....T.......$PnP|
00000200
```

### Testing different options with /etc/pve/qemu-server/100.conf

```shell
# No effect
hostpci0: 0000:03:00.0,pcie=1
# No effect
hostpci0: 0000:03:00.0,pcie=1,rombar=0 
# No effect
hostpci0: 0000:03:00.0,pcie=1,rombar=0,sub-device-id=0x0612,sub-vendor-id=0x1b21 
# No effect (copied ROM to  /usr/share/kvm/)
hostpci0: 0000:03:00.0,pcie=1,rombar=0,romfile=asm1061.rom
# VM refused to start
hostpci0: 0000:03:00.0,pcie=1,x-vga=on

# Outcome of tests was observed reading dmesg on VM, in all cases the result was the same
[    0.436960] pci 0000:01:00.0: [1b21:0612] type 7f class 0xffffff conventional PCI
[    0.437097] pci 0000:01:00.0: unknown header type 7f, ignoring device
```

### Inspecting lspci

```shell
lspci -nnvv
# ASM1061 after clean reboot
03:00.0 SATA controller [0106]: ASMedia Technology Inc. ASM1062 Serial ATA Controller [1b21:0612] (rev 02) (prog-if 01 [AHCI 1.0])
	Subsystem: ASMedia Technology Inc. ASM1061/ASM1062 Serial ATA Controller [1b21:1060]
	Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 64 bytes
	Interrupt: pin A routed to IRQ 255
	IOMMU group: 12
	Region 0: I/O ports at 3050 [size=8]
	Region 1: I/O ports at 3040 [size=4]
	Region 2: I/O ports at 3030 [size=8]
	Region 3: I/O ports at 3020 [size=4]
	Region 4: I/O ports at 3000 [size=32]
	Region 5: Memory at 80510000 (32-bit, non-prefetchable) [size=512]
	Expansion ROM at 80500000 [disabled] [size=64K]
	Capabilities: [50] MSI: Enable- Count=1/1 Maskable- 64bit-
		Address: 00000000  Data: 0000
	Capabilities: [78] Power Management version 3
		Flags: PMEClk- DSI- D1- D2- AuxCurrent=0mA PME(D0-,D1-,D2-,D3hot-,D3cold-)
		Status: D3 NoSoftRst- PME-Enable- DSel=0 DScale=0 PME-
	Capabilities: [80] Express (v2) Legacy Endpoint, MSI 00
		DevCap:	MaxPayload 512 bytes, PhantFunc 0, Latency L0s <1us, L1 <8us
			ExtTag- AttnBtn- AttnInd- PwrInd- RBE+ FLReset-
		DevCtl:	CorrErr- NonFatalErr- FatalErr- UnsupReq-
			RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop+
			MaxPayload 256 bytes, MaxReadReq 512 bytes
		DevSta:	CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
		LnkCap:	Port #1, Speed 5GT/s, Width x1, ASPM not supported
			ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp-
		LnkCtl:	ASPM Disabled; RCB 64 bytes, Disabled- CommClk+
			ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
		LnkSta:	Speed 5GT/s, Width x1
			TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
		DevCap2: Completion Timeout: Range ABC, TimeoutDis+ NROPrPrP- LTR-
			 10BitTagComp- 10BitTagReq- OBFF Not Supported, ExtFmt- EETLPPrefix-
			 EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
			 FRS-
			 AtomicOpsCap: 32bit- 64bit- 128bitCAS-
		DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- LTR- 10BitTagReq- OBFF Disabled,
			 AtomicOpsCtl: ReqEn-
		LnkCtl2: Target Link Speed: 5GT/s, EnterCompliance- SpeedDis-
			 Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
			 Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
		LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete- EqualizationPhase1-
			 EqualizationPhase2- EqualizationPhase3- LinkEqualizationRequest-
			 Retimer- 2Retimers- CrosslinkRes: unsupported
	Capabilities: [100 v1] Virtual Channel
		Caps:	LPEVC=0 RefClk=100ns PATEntryBits=1
		Arb:	Fixed- WRR32- WRR64- WRR128-
		Ctrl:	ArbSelect=Fixed
		Status:	InProgress-
		VC0:	Caps:	PATOffset=00 MaxTimeSlots=1 RejSnoopTrans-
			Arb:	Fixed- WRR32- WRR64- WRR128- TWRR128- WRR256-
			Ctrl:	Enable+ ID=0 ArbSelect=Fixed TC/VC=ff
			Status:	NegoPending- InProgress-
	Kernel driver in use: vfio-pci
	Kernel modules: ahci


# ASM1061 after trying to start VM
03:00.0 SATA controller: ASMedia Technology Inc. ASM1062 Serial ATA Controller (rev 02) (prog-if 01 [AHCI 1.0])
	Subsystem: ASMedia Technology Inc. ASM1061/ASM1062 Serial ATA Controller
	!!! Unknown header type 7f
	Interrupt: pin ? routed to IRQ 18
	IOMMU group: 12
	Region 0: I/O ports at 3050 [size=8]
	Region 1: I/O ports at 3040 [size=4]
	Region 2: I/O ports at 3030 [size=8]
	Region 3: I/O ports at 3020 [size=4]
	Region 4: I/O ports at 3000 [size=32]
	Region 5: Memory at 80510000 (32-bit, non-prefetchable) [size=512]
	Expansion ROM at 80500000 [disabled] [size=64K]
	Kernel driver in use: vfio-pci
	Kernel modules: ahci
```
`DevCap: ... FLReset-` shows that ASM1061 on Terramaster F4-424 Pro doesn't support Function Level Reset (FLR). Without this capability the hypervisor cannot properly reset the ASMedia controller.


### Journalctl messages when trying to start VM with Disks connected

![Troubleshooting](/assets/img/posts/pve-asm-pt/pve-asm-pt06.jpg){: width="1205" height="701" .center}

The ASM1061 seems to crash when the virtual machine tries to utilize it. Simultaneously the PCI bridge (at address `00:1c.6`), where ASM1061 is connected, reports a "broken device, retraining non-functional downstream link at 2.5GT/s" error.

```shell
[  986.073584] pcieport 0000:00:1c.6: broken device, retraining non-functional downstream link at 2.5GT/s
[  987.073607] pcieport 0000:00:1c.6: retraining failed
[  988.265580] pcieport 0000:00:1c.6: broken device, retraining non-functional downstream link at 2.5GT/s
[  989.266692] pcieport 0000:00:1c.6: retraining failed
[  989.266719] vfio-pci 0000:03:00.0: not ready 1023ms after bus reset; waiting
[  990.313715] vfio-pci 0000:03:00.0: not ready 2047ms after bus reset; waiting
[  992.425904] vfio-pci 0000:03:00.0: not ready 4095ms after bus reset; waiting
[  996.970032] vfio-pci 0000:03:00.0: not ready 8191ms after bus reset; waiting
[ 1005.674397] vfio-pci 0000:03:00.0: not ready 16383ms after bus reset; waiting
[ 1022.571103] vfio-pci 0000:03:00.0: not ready 32767ms after bus reset; waiting
[ 1058.412561] vfio-pci 0000:03:00.0: not ready 65535ms after bus reset; giving up
[ 1059.503624] pcieport 0000:00:1c.6: broken device, retraining non-functional downstream link at 2.5GT/s
[ 1060.505639] pcieport 0000:00:1c.6: retraining failed
[ 1061.676721] pcieport 0000:00:1c.6: broken device, retraining non-functional downstream link at 2.5GT/s
[ 1062.678727] pcieport 0000:00:1c.6: retraining failed
[ 1062.678754] vfio-pci 0000:03:00.0: not ready 1023ms after bus reset; waiting
[ 1063.724798] vfio-pci 0000:03:00.0: not ready 2047ms after bus reset; waiting
[ 1065.836882] vfio-pci 0000:03:00.0: not ready 4095ms after bus reset; waiting
[ 1070.189080] vfio-pci 0000:03:00.0: not ready 8191ms after bus reset; waiting
[ 1078.893379] vfio-pci 0000:03:00.0: not ready 16383ms after bus reset; waiting
[ 1095.790122] vfio-pci 0000:03:00.0: not ready 32767ms after bus reset; waiting
[ 1132.143635] vfio-pci 0000:03:00.0: not ready 65535ms after bus reset; giving up
[ 1132.147053] vfio-pci 0000:03:00.0: Unable to change power state from D0 to D3hot, device inaccessible
```

### Kernel CMD Configurations
![GRUB](/assets/img/posts/pve-asm-pt/pve-asm-pt07.jpg){: width="1205" height="701" .center}
_Kernel parameters_

```shell
# IOMMU is actually enabled by default since Proxmox kernels 6.8 
GRUB_CMDLINE_LINUX="intel_iommu=on"
# Passthrough
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt"
# Disable 
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt, pcie_aspm=off"
```
