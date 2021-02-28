---
layout: post
title: Proxmox GPU Passthough to Ubuntu VM
date: 2018-12-22 00:00:00 +0300
description: Proxmox GPU Passthrough on Ubuntu 18.04 # Add post description (optional)
img:  # Add image post (optional)
tags: [Proxmox, GPU] # add tag
---

For this project I am using a repurposed old desktop, that is now a whitebox proxmox cluster node.  I am using Proxmox 5.3-5 at the time of install.

![pve node]( https://nopresearcher.github.io/assets/images/pve-node.png "PVE Node")

Tech Specs
* MSI 970A-G46 AM3+/AM3 AMD 970
* AMD FX-6100 Zambezi 6-Core 3.3 GHz Socet AM3+
* 2 x 8GB G.Skill Ripjaws X Series 240-pin DDR3 1866 RAM
* Intel Quad NIC
* GeForce GTX 970
* SSD 240GB

Boot into the BIOS and ensure the following is supported and enabled.

* Ensure VT-d is enabled
* Interrupt remapping is supported or an option for IOMMU is enabled
* UEFI BIOS
* Ensure another video card is enabled.  For me it was the integrated on board graphics card.

## Configure Proxmox node

Edit GRUB to enable IOMMU.
```bash
nano /etc/default/grub

GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on efifb=off"

update-grub
```

Verify that IOMMU is enabled.

```bash
dmesg | grep -e DMAR -e IOMMU

[    0.000000] DMAR: IOMMU enabled
[    1.219719] AMD-Vi: Found IOMMU at 0000:00:00.2 cap 0x40
```

Add the Nvidia and Nouveau drivers to the blacklist to prevent them from being loaded by Proxmox.

```bash
echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "blacklist radeon" >> /etc/modprobe.d/blacklist.conf

update-initramfs -u
```

Verify which driver has been loaded, below it is still the  nouveau driver.  The point is that it is recognized by Proxmox.

```bash
lspci -v

01:00.0 VGA compatible controller: NVIDIA Corporation GM204 [GeForce GTX 970] (rev a1) (prog-if 00 [VGA controller])
    Subsystem: NVIDIA Corporation GM204 [GeForce GTX 970]
    Flags: bus master, fast devsel, latency 0, IRQ 70, NUMA node 0
    Memory at fa000000 (32-bit, non-prefetchable) [size=16M]
    Memory at c0000000 (64-bit, prefetchable) [size=256M]
    Memory at d0000000 (64-bit, prefetchable) [size=32M]
    I/O ports at e000 [size=128]
    Expansion ROM at 000c0000 [disabled] [size=128K]
    Capabilities: [60] Power Management version 3
    Capabilities: [68] MSI: Enable+ Count=1/1 Maskable- 64bit+
    Capabilities: [78] Express Legacy Endpoint, MSI 00
    Capabilities: [100] Virtual Channel
    Capabilities: [250] Latency Tolerance Reporting
    Capabilities: [258] L1 PM Substates
    Capabilities: [128] Power Budgeting <?>
    Capabilities: [600] Vendor Specific Information: ID=0001 Rev=1 Len=024 <?>
    Capabilities: [900] #19
    Kernel driver in use: nouveau
    Kernel modules: nvidiafb, nouveau

01:00.1 Audio device: NVIDIA Corporation GM204 High Definition Audio Controller (rev a1)
	Subsystem: NVIDIA Corporation GM204 High Definition Audio Controller
	Flags: bus master, fast devsel, latency 0, IRQ 68, NUMA node 0
	Memory at fb080000 (32-bit, non-prefetchable) [size=16K]
	Capabilities: [60] Power Management version 3
	Capabilities: [68] MSI: Enable- Count=1/1 Maskable- 64bit+
	Capabilities: [78] Express Endpoint, MSI 00
	Kernel driver in use: snd_hda_intel
	Kernel modules: snd_hda_intel
```

Add Virtual Function IO (vfio) kernel modules to load at boot time.

```bash
nano /etc/modules

vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

Determine the GeForce GTX 970 address and IDs.  As seen above in the lspci output it was 01:00.

```bash
lspci -n -s 01:00


    01:00.0 0300: 10de:13c2 (rev a1)
    01:00.1 0403: 10de:0fbb (rev a1)
```

Assign the GeForce GTX 970 to the Virtual Function IO (vfio).

```bash
echo "options vfio-pci ids=10de:13c2, 10de:0fbb" > /etc/modprobe.d/vfio.conf
```

If the motherbaord does nothave an option for IOMMU interrupt remapping, you may have to enable it.

```bash
echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf
```

Reboot the Proxmox node.  Verify that the GeForce GTX 970 is loaded and using the vfio-pci Kernel dirver.  Ensure that it has capabilities listed as well.

```bash
lspci -v

01:00.0 VGA compatible controller: NVIDIA Corporation GM204 [GeForce GTX 970] (rev a1) (prog-if 00 [VGA controller])
        Subsystem: NVIDIA Corporation GM204 [GeForce GTX 970]
        Flags: bus master, fast devsel, latency 0, IRQ 70, NUMA node 0
        Memory at f4000000 (32-bit, non-prefetchable) [size=16M]
        Memory at c0000000 (64-bit, prefetchable) [size=256M]
        Memory at d0000000 (64-bit, prefetchable) [size=32M]
        I/O ports at e000 [size=128]
        Expansion ROM at 000c0000 [disabled] [size=128K]
        Capabilities: [60] Power Management version 3
        Capabilities: [68] MSI: Enable+ Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Legacy Endpoint, MSI 00
        Capabilities: [100] Virtual Channel
        Capabilities: [250] Latency Tolerance Reporting
        Capabilities: [258] L1 PM Substates
        Capabilities: [128] Power Budgeting <?>
        Capabilities: [600] Vendor Specific Information: ID=0001 Rev=1 Len=024 <?>
        Kernel driver in use: vfio-pci
        Kernel modules: nvidiafb, nouveau

01:00.1 Audio device: NVIDIA Corporation GM204 High Definition Audio Controller (rev a1)
        Subsystem: NVIDIA Corporation GM204 High Definition Audio Controller
        Flags: bus master, fast devsel, latency 0, IRQ 69, NUMA node 0
        Memory at f5080000 (32-bit, non-prefetchable) [size=16K]
        Capabilities: [60] Power Management version 3
        Capabilities: [68] MSI: Enable- Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Endpoint, MSI 00
        Kernel driver in use: vfio-pci
        Kernel modules: snd_hda_intel
```


## Create Ubuntu 18.04 VM

Create VM with UEFI bios

![pve vm]( https://nopresearcher.github.io/assets/images/pve-vm-hardware.png "PVE VM")

Start the VM without the graphics card and enable SSH.  Once SSH is enabled, only use SSH to interact it with it to prevent and display issues.  You will not be able to interact with it via VNC, noVNC, etc.


![pve vm info]( https://nopresearcher.github.io/assets/images/pve-vm-info.png "PVE VM infow")

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl status ssh
```

Once the VM is installed and SSH is enabled. Power down the VM and add the graphics card to the VM.  On the Proxmox host edit the vm conf file.

hostpci0: is the Proxmox host
01:00 is the addess of the GeForce GTX 970 on the Proxmox host
x-vga=on enables vfio-vga device support

```bash
nano /etc/pve/qemu-server/(VMID).conf
hostpci0: 01:00,x-vga=on
```

Add the repository for the nvidia driver.  I tried using the standard repos however, those drivers did not work.

```bash
sudo apt-add-repository ppa:xorg-edgers/ppa
sudo apt-get install nvidia-375 nvidia-libopencl1-375 
```

Blacklist the nouvea driver to prevent it from being used for the graphics card.

```bash
sudo bash -c "echo blacklist nouveau > /etc/modprobe.d/blacklist-nvidia-nouveau.conf"

sudo bash -c "echo options nouveau modeset=0 >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf"

sudo update-initramfs -u
```

Edit GRUB to prevent issues with booting.

```bash
sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
# and change it to
GRUB_CMDLINE_LINUX_DEFAULT="nomodeset quiet splash"

sudo update-grub
```

At boot ensure the card is found the driver can communicate with the card.

```bash
sudo nano /etc/rc.local


#!/bin/bash

/sbin/modprobe nvidia

if [ "$?" -eq 0 ]; then

# Count the number of NVIDIA controllers found.
N3D=`/lspci | grep -i NVIDIA | grep "3D controller" | wc -l`
NVGA=`lspci | grep -i NVIDIA | grep "VGA compatible controller" | wc -l`

N=`expr $N3D + $NVGA - 1`
for i in `seq 0 $N`; do
mknod -m 666 /dev/nvidia$i c 195 $i;
done

mknod -m 666 /dev/nvidiactl c 195 255

else
exit 1
fi

exit 0
```

Now Reboot the VM.  You should have a working GeForce GTX 970 in an Ubunutu 18.04 VM.
