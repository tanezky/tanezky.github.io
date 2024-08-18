---
title: Proxmox - HDD Passthrough Using SATA Controllers
author: tanezky
date: 2024-08-16 20:00:00 +/-TTTT
description: Testing ASM1061 passthrough on Terramaster F4-424 Pro
categories: [NAS]
tags: [NAS,Terramaster,F4-424 Pro, Proxmox, Truenas, Virtualization]
image: /assets/img/posts/pve-asm-pt/pve-asm-pt-post.jpg
---

> **tl;dr: ASM1061 did not work with hardware passthrough**.
{: .prompt-info }

## Just for Fun: Truenas on Proxmox
While running TrueNAS on top of Proxmox might not make any sense, it's an interesting experiment to explore hard disk passthrough and see how well TrueNAS recognizes the virtualized hardware. Plus, it offers a chance to test if the ASMedia ASM1061 controller on the TerraMaster F4-424 Pro plays nicely with passthrough.

## Installing TrueNAS
### Getting ISO
![Get TrueNAS ISO](/assets/img/posts/pve-asm-pt/pve-asm-pt01.jpg){: width="1307" height="574" .center}

1. local storage
2. ISO Images
3. Download from URL
4. Get latest url from Truenas [download page](https://www.truenas.com/download-truenas-scale/)
5. Query URL to get `File name` populated
6. Hash algorithm: SHA-256, copy paste checksum from Truenas download page
7. Download

### Create Virtual Machine

If not mentioned, use default value.

**General**
- Node: aurum-vault (your node name)
- Name: truenas

**OS**
- ISO image: Select TruenasScale ISO

**System**
- Machine: q35
- BIOS: OVMF(UEFI)
- Add EFI Disk
- EFI Storage: local-lvm

**Disks**
- Storage: local-lvm
- Disk size (GiB): 32

**CPU**
- Cores: 4
- Type: x86-64-v2-AES

**Memory**
- Memory (MiB): 16384

### Starting Installation
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
lspci -k
00:00.0 Host bridge: Intel Corporation Device 4617
        DeviceName: Onboard - Other
        Subsystem: Intel Corporation Device 7270
        Kernel driver in use: igen6_edac
        Kernel modules: igen6_edac
00:02.0 VGA compatible controller: Intel Corporation Alder Lake-N [UHD Graphics]
        DeviceName: Onboard - Video
        Subsystem: Intel Corporation Alder Lake-N [UHD Graphics]
        Kernel driver in use: i915
        Kernel modules: i915, xe
00:14.0 USB controller: Intel Corporation Alder Lake-N PCH USB 3.2 xHCI Host Controller
        DeviceName: Onboard - Other
        Subsystem: Intel Corporation Alder Lake-N PCH USB 3.2 xHCI Host Controller
        Kernel driver in use: xhci_hcd
        Kernel modules: xhci_pci
00:14.2 RAM memory: Intel Corporation Alder Lake-N PCH Shared SRAM
        DeviceName: Onboard - Other
        Subsystem: Intel Corporation Alder Lake-N PCH Shared SRAM
00:16.0 Communication controller: Intel Corporation Alder Lake-N PCH HECI Controller
        DeviceName: Onboard - Other
        Subsystem: Intel Corporation Alder Lake-N PCH HECI Controller
        Kernel driver in use: mei_me
        Kernel modules: mei_me
00:17.0 SATA controller: Intel Corporation Device 54d3
        DeviceName: Onboard - SATA
        Subsystem: Intel Corporation Device 7270
        Kernel driver in use: ahci
        Kernel modules: ahci
00:1c.0 PCI bridge: Intel Corporation Device 54ba
        Subsystem: Intel Corporation Device 7270
        Kernel driver in use: pcieport
00:1c.3 PCI bridge: Intel Corporation Device 54bb
        Subsystem: Intel Corporation Device 7270
        Kernel driver in use: pcieport
00:1c.6 PCI bridge: Intel Corporation Device 54be
        Subsystem: Intel Corporation Device 7270
        Kernel driver in use: pcieport
00:1d.0 PCI bridge: Intel Corporation Device 54b0
        Subsystem: Intel Corporation Device 7270
        Kernel driver in use: pcieport
00:1f.0 ISA bridge: Intel Corporation Alder Lake-N PCH eSPI Controller
        DeviceName: Onboard - Other
        Subsystem: Intel Corporation Alder Lake-N PCH eSPI Controller
00:1f.3 Audio device: Intel Corporation Alder Lake-N PCH High Definition Audio Controller
        DeviceName: Onboard - Sound
        Subsystem: Intel Corporation Alder Lake-N PCH High Definition Audio Controller
        Kernel driver in use: snd_hda_intel
        Kernel modules: snd_hda_intel, snd_sof_pci_intel_tgl
00:1f.4 SMBus: Intel Corporation Device 54a3
        DeviceName: Onboard - Other
        Subsystem: Intel Corporation Device 7270
        Kernel driver in use: i801_smbus
        Kernel modules: i2c_i801
00:1f.5 Serial bus controller: Intel Corporation Device 54a4
        DeviceName: Onboard - Other
        Subsystem: Intel Corporation Device 7270
        Kernel driver in use: intel-spi
        Kernel modules: spi_intel_pci
01:00.0 Ethernet controller: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller (rev 05)
        Subsystem: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller
        Kernel driver in use: r8169
        Kernel modules: r8169
02:00.0 Ethernet controller: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller (rev 05)
        Subsystem: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller
        Kernel driver in use: r8169
        Kernel modules: r8169
03:00.0 SATA controller: ASMedia Technology Inc. ASM1062 Serial ATA Controller (rev 02)
        Subsystem: ASMedia Technology Inc. ASM1061/ASM1062 Serial ATA Controller
        Kernel driver in use: ahci
        Kernel modules: ahci
04:00.0 Non-Volatile memory controller: Samsung Electronics Co Ltd NVMe SSD Controller 980
        Subsystem: Samsung Electronics Co Ltd NVMe SSD Controller 980 (DRAM-less)
        Kernel driver in use: nvme
        Kernel modules: nvme
```

#### Add PCI Device
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

## Troubleshooting ASM1061
![Troubleshooting](/assets/img/posts/pve-asm-pt/pve-asm-pt06.jpg){: width="1205" height="701" .center}
Unfortunately when I attempted to repeat the Add PCI Device steps for the ASM1061 controller (at address `03:00.0`) it failed to work as expected.

After trying various troubleshooting techniques I concluded that the ASM1061 seems to crash when the virtual machine tries to utilize it. Simultaneously the PCI bridge at address `00:1c.6` reports a "broken device, retraining non-functional downstream link at 2.5GT/s" error.

When the VM attempts to use the ASM1061 the driver changes from `ahci` to `vfio-pci`. While these are my initial observations the root cause could lie elsewhere. It could be an issue with how the UEFI initializes the ASM1061, a firmware problem within the 1061 itself or even a hardware flaw.

Fortunately, since I don't require hardware passthrough for HDDs to virtual machines, these issues won't directly impact my usage. However, it remains to see for any other potential complications as I continue experimenting with the system.

### Results

### Inspecting lspci

```shell
# asm1061 after Proxmox reboot
03:00.0 SATA controller: ASMedia Technology Inc. ASM1062 Serial ATA Controller (rev 02) (prog-if 01 [AHCI 1.0])
        Subsystem: ASMedia Technology Inc. ASM1061/ASM1062 Serial ATA Controller
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
                DevCap: MaxPayload 512 bytes, PhantFunc 0, Latency L0s <1us, L1 <8us
                        ExtTag- AttnBtn- AttnInd- PwrInd- RBE+ FLReset-
                DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                        RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop+
                        MaxPayload 256 bytes, MaxReadReq 512 bytes
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkCap: Port #1, Speed 5GT/s, Width x1, ASPM not supported
                        ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp-
                LnkCtl: ASPM Disabled; RCB 64 bytes, Disabled- CommClk+
                        ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
                LnkSta: Speed 5GT/s, Width x1
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
                Caps:   LPEVC=0 RefClk=100ns PATEntryBits=1
                Arb:    Fixed- WRR32- WRR64- WRR128-
                Ctrl:   ArbSelect=Fixed
                Status: InProgress-
                VC0:    Caps:   PATOffset=00 MaxTimeSlots=1 RejSnoopTrans-
                        Arb:    Fixed- WRR32- WRR64- WRR128- TWRR128- WRR256-
                        Ctrl:   Enable+ ID=0 ArbSelect=Fixed TC/VC=ff
                        Status: NegoPending- InProgress-
        Kernel driver in use: vfio-pci
        Kernel modules: ahci


# asm1061 after trying to start VM
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

### Journalctl messages when trying to start VM

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
### Configurations and useful commands
![GRUB](/assets/img/posts/pve-asm-pt/pve-asm-pt07.jpg){: width="1205" height="701" .center}
_Kernel parameters_

```shell
# Different kernel parameters which were tried
GRUB_CMDLINE_LINUX="intel_iommu=on"
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt"
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt, pcie_aspm=off"

# Get device vendor and model
lspci -nn
03:00.0 SATA controller [0106]: ASMedia Technology Inc. ASM1062 Serial ATA Controller [1b21:0612] (rev 02)

# Part 1: Force asm1061 to use vfio-pci driver, blacklist ahci
cat /etc/modprobe.d/pve-blacklist.conf 
blacklist ahci

# Part 2: Instruct kernel to use vfio-pci driver for asm1061
cat /etc/modprobe.d/vfio.conf 
options vfio-pci ids=1b21:0612

# Query which kernel module and driver device is using
lspci -k -d 1b21:0612
03:00.0 SATA controller: ASMedia Technology Inc. ASM1062 Serial ATA Controller (rev 02)
        Subsystem: ASMedia Technology Inc. ASM1061/ASM1062 Serial ATA Controller
        Kernel driver in use: vfio-pci
        Kernel modules: ahci
```