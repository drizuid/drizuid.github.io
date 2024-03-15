---
title: PC for my kids
author: Will
type: posts
date: 2017-01-11T21:14:00+00:00
url: /2017/01/pc-for-my-kids/
categories:
  - kvm
  - Linux
  - Operating Systems
  - virt
summary: A little background. I have a 7 and 4 year old who like playing on the computer. Typically they play things like animaljam or nickjr games, very simple requirements, minimal hardware needs. One day, the horror that is minecraft crept its way into my home and suddenly, graphics (seriously? in minecraft, it's a bunch of blocks…) mattered.
---
A little background. I have a 7 and 4 year old who like playing on the computer. Typically they play things like animaljam or nickjr games, very simple requirements, minimal hardware needs. One day, the horror that is minecraft crept its way into my home and suddenly, graphics (seriously? in minecraft, it's a bunch of blocks…) mattered.

At this time there was my old pc with a GTX 460, an SSD, 16G ram, and a nice CPU running windows 10. Beside this system was a Dell Latitude e6510 with a regular 5400 rpm hdd, 4G ram, and a trash cpu running Linux Mint. On minimal graphics settings and fullscreen, minecraft was just almost playable on the laptop, but there was always a fight for the big computer.

Being tired of tracking who was on it last, I tried to think up an idea to resolve this issue while creating a fun project for the kids and I. I came up with a plan to build one system, stick ESXi on it and use pci passthrough to make 2 "thin clients." The 7 year old would assist in building the PC and the 4 year old would affix the typical stickers that come with systems as she wished. &nbsp;Here begins the story.

So I began doing some research, obviously there are some requirements such as VT-d and VT-x to properly support pci passthrough. After thorough research on anandtech, toms hardware, and various sites about doing similar projects, I decided that I knew better than all these people and went rogue.

Quick note: one of my main sources as I worked on this project was https://forums.linuxmint.com/viewtopic.php?f=231&t=212692. Thanks Powerhouse!

I wanted the newest shit because it's new and I didn't want the old proven-to-work stuff. Thus, i made the following purchases using my budget of $400.

  * Motherboard: ASRock Micro ATX H170M PRO4 – 89.99
  * CPU: Intel 3.7GHz Core i3-6100 117.81
  * RAM: Patriot Extreme Performance 8G PC4-25600 (3200MHz) Viper Elite Series 49.99
  * Video: 2x MSI AMD Radeon R7 240 2GB DDR3 Low Profile PCIE 45.99 each
  * Audio: 2x Sabrent USB External Stero Sound Adapter (C-Media) 6.99 each
  * Storage: 2x Adata SU800 128G 3D-NAND 2.5″ SATA3 SSD 44.99 each  
    _Well, to be honest, since i wasn't sure it would work, i first purchased only 1 video card and no SSDs. I also hadn't purchased the USB audio adapters yet.&nbsp;_

Moving on, I installed esxi 6.5 and went to pass through things. video passed through flawlessly, and then i realized that every single usb port on the motherboard was under 1 controller which i couldn't pass through on esxi if i wanted to. &nbsp;I tried some vmx trickery to get the HID devices passed through directly, but failed miserably.

Well, I've been a linux lover since '96, why not try linux kvm, it had surely matured since my last try years before. In comes debian linux, dependencies and qemu compiled from git. Using a raw disk image on a 5400rpm sata2 disk worked! it was impossibly slow, but everything worked, pci passthrough was successful and i was happy. ( i will post how i did everything a little later, we're still in the story stage) I ordered the 2 SSDs and the second video card the same day.

Installation of hardware complete, time to get the relevant info and write a bash script. Let's look at some actual information here.

```Shell
-[0000:00]-+-00.0 Intel Corporation Skylake Host Bridge/DRAM Registers [8086:190f]
	   +-01.0-[01]--+-00.0 Advanced Micro Devices, Inc. [AMD/ATI] Oland PRO [Radeon R7 240/340] 
[1002:6613]
	   |            \-00.1 Advanced Micro Devices, Inc. [AMD/ATI] Cape Verde/Pitcairn HDMI Audio [Radeon HD 7700/7800 Series] [1002:aab0]
	   +-02.0 Intel Corporation HD Graphics 530 [8086:1912]
	   +-14.0 Intel Corporation Sunrise Point-H USB 3.0 xHCI Controller [8086:a12f]
	   +-14.2 Intel Corporation Sunrise Point-H Thermal subsystem [8086:a131]
	   +-16.0 Intel Corporation Sunrise Point-H CSME HECI #1 [8086:a13a]
	   +-17.0 Intel Corporation Sunrise Point-H SATA controller [AHCI mode] [8086:a102]
	   +-1b.0-[02]--
	   +-1c.0-[03]--
	   +-1c.4-[04]--+-00.0 Advanced Micro Devices, Inc. [AMD/ATI] Oland PRO [Radeon R7 240/340] 
[1002:6613]
	   |            \-00.1 Advanced Micro Devices, Inc. [AMD/ATI] Cape Verde/Pitcairn HDMI Audio [Radeon HD 7700/7800 Series] [1002:aab0]
	   +-1d.0-[05]--
	   +-1f.0 Intel Corporation Sunrise Point-H LPC Controller [8086:a144]
	   +-1f.2 Intel Corporation Sunrise Point-H PMC [8086:a121]
	   +-1f.3 Intel Corporation Sunrise Point-H HD Audio [8086:a170]
	   +-1f.4 Intel Corporation Sunrise Point-H SMBus [8086:a123]
	   \-1f.6 Intel Corporation Ethernet Connection (2) I219-V [8086:15b8]
```

Well, as I mentioned before, 1 USB controller so we'll be passing through individual devices. You can also see the two Radeon R7 240s on here with their corresponding bits of info.  
Next we need to ensure that our devices are not being used to the host system. To accomplish this, I blacklist the radeon driver in /etc/modprobe.d/blacklist.conf and include it in /etc/initramfs-tools/modules. Please don't use pci-stub, there's really just no reason to do so anymore.

```Shell
01:00.0 0300: 1002:6613 (rev 87)
        Subsystem: 1462:8094
        Kernel driver in use: vfio-pci
        Kernel modules: radeon
01:00.1 0403: 1002:aab0
        Subsystem: 1462:aab0
        Kernel driver in use: vfio-pci
        Kernel modules: snd_hda_intel
04:00.0 0300: 1002:6613 (rev 87)
        Subsystem: 1462:8094
        Kernel driver in use: vfio-pci
        Kernel modules: radeon
04:00.1 0403: 1002:aab0
        Subsystem: 1462:aab0
        Kernel driver in use: vfio-pci
        Kernel modules: snd_hda_intel
```

I then downloaded the latest ovmf code from git and modified /etc/libvirt/qemu.conf to have

```Shell
nvram = [
        "/srv/ovmf-x64/OVMF_CODE-pure-efi.fd:/srv/ovmf-x64/OVMF_VARS-pure-efi.fd"
]
```

My grub command line is as follows:

`GRUB_CMDLINE_LINUX_DEFAULT="rd.driver.pre=vfio-pci quiet splash intel_iommu=on vfio_iommu_type1.allow_unsafe_interrupts=1 iommu=pt"`

There are some additional items you can pass when using the 170 chipset or a 915 onboard graphics, both of which I am, but i noticed no difference using them vs not using them and thus removed the cmdline options.

I modified /etc/modules to include the following

```Shell
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
vhost-net
kvm
kvm_intel
```

/etc/vfio-pci1.cfg

```Shell
0000:01:00.0
0000:01:00.1
```

/etc/vfio-pci2.cfg

```Shell
0000:04:00.0
0000:04:00.1
```

/etc/network/interfaces

```Shell
auto br0
iface br0 inet dhcp
  bridge_ports enp0s31f6
  bridge_stp off
  bridge_maxwait 0
  bridge_fd 0
  post-up ip link set dev enp0s31f6 mtu 9000
```

in /etc/udev/rules.d/70-persistent-net.rules i have

`SUBSYSTEM=="net", ACTION=="add", KERNEL=="tap*", ATTR{mtu}="9000"`

My launch script for vm1 is

```bash
#!/bin/bash

configfile=/etc/vfio-pci1.cfg
vmname="system1"

vfiobind() {
   dev="$1"
        vendor=$(cat /sys/bus/pci/devices/$dev/vendor)
        device=$(cat /sys/bus/pci/devices/$dev/device)
        if [ -e /sys/bus/pci/devices/$dev/driver ]; then
                echo $dev > /sys/bus/pci/devices/$dev/driver/unbind
        fi
        echo $vendor $device > /sys/bus/pci/drivers/vfio-pci/new_id

}


if ps -A | grep -q $vmname; then
   echo "$vmname is already running."
   exit 1

else

   cat $configfile | while read line;do
   echo $line | grep ^# >/dev/null 2>&1 && continue
      vfiobind $line
   done


if [ ! -f /tmp/system1.fd ]; then
  cp /srv/ovmf-x64/OVMF_VARS-pure-efi.fd /tmp/system1.fd
fi

taskset -c 0,3 qemu-system-x86_64 \
  -name $vmname,process=$vmname \
  -serial none \
  -parallel none \
  -machine type=q35,accel=kvm \
  -cpu host,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time \
  -smp 2,sockets=1,cores=1,threads=2 \
  -enable-kvm \
  -m 3G \
  -mem-prealloc \
  -balloon none \
  -rtc clock=host,base=localtime \
  -vga none \
  -nographic \
  -usb -usbdevice host:040b:2000 -usbdevice host:046d:c01e -usbdevice host:0d8c:0014 -usbdevice host:046d:0819 \
  -device ioh3420,bus=pcie.0,addr=1c.0,multifunction=on,port=1,chassis=1,id=root.1 \
  -device vfio-pci,host=01:00.0,bus=root.1,addr=00.0,multifunction=on,romfile=/srv/rom1.rom \
  -device vfio-pci,host=01:00.1,bus=root.1 \
  -drive if=pflash,format=raw,readonly,file=/srv/ovmf-x64/OVMF_CODE-pure-efi.fd \
  -drive if=pflash,format=raw,file=/tmp/system1.fd \
  -boot order=c \
  -device virtio-scsi-pci,id=scsi \
  -drive file=/dev/sdc1,id=disk0,if=none,format=raw,cache=none,aio=native \
  -device scsi-hd,drive=disk0 \
  -netdev type=tap,id=net0,ifname=tap0,vhost=on \
  -device virtio-net-pci,netdev=net0,mac=52:54:00:ea:c8:8d

#  -mem-path /dev/hugepages1 \

exit 0
fi
```

My launch script for vm2 is

```bash
#!/bin/bash

configfile=/etc/vfio-pci2.cfg
vmname="system2"

vfiobind() {
   dev="$1"
        vendor=$(cat /sys/bus/pci/devices/$dev/vendor)
        device=$(cat /sys/bus/pci/devices/$dev/device)
        if [ -e /sys/bus/pci/devices/$dev/driver ]; then
                echo $dev > /sys/bus/pci/devices/$dev/driver/unbind
        fi
        echo $vendor $device > /sys/bus/pci/drivers/vfio-pci/new_id

}


if ps -A | grep -q $vmname; then
   echo "$vmname is already running."
   exit 1

else

   cat $configfile | while read line;do
   echo $line | grep ^# >/dev/null 2>&1 && continue
      vfiobind $line
   done


if [ ! -f /tmp/system2.fd ]; then
  cp /srv/ovmf-x64/OVMF_VARS-pure-efi.fd /tmp/system2.fd
fi

taskset -c 1,2 qemu-system-x86_64 \
  -name $vmname,process=$vmname \
  -serial none \
  -parallel none \
  -machine type=q35,accel=kvm \
  -cpu host,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time \
  -smp 2,sockets=1,cores=1,threads=2 \
  -enable-kvm \
  -m 3G \
  -mem-prealloc \
  -balloon none \
  -rtc clock=host,base=localtime \
  -vga none \
  -nographic \
  -usb -usbdevice host:04f2:0939 -usbdevice host:04f2:0833 \
  -device ioh3420,bus=pcie.0,addr=1c.0,multifunction=on,port=1,chassis=1,id=root.1 \
  -device vfio-pci,host=04:00.0,bus=root.1,addr=00.0,multifunction=on,romfile=/srv/rom2.rom \
  -device vfio-pci,host=04:00.1,bus=root.1 \
  -drive if=pflash,format=raw,readonly,file=/srv/ovmf-x64/OVMF_CODE-pure-efi.fd \
  -drive if=pflash,format=raw,file=/tmp/system2.fd \
  -boot order=c \
  -device virtio-scsi-pci,id=scsi \
  -drive file=/dev/sdd1,id=disk0,if=none,format=raw,cache=none,aio=native \
  -device scsi-hd,drive=disk0 \
  -netdev type=tap,id=net0,ifname=tap1,vhost=on \
  -device virtio-net-pci,netdev=net0,mac=52:54:00:ea:c8:8e

#  -mem-path /dev/hugepages2 \

exit 0
fi
```

i create a screen session for each system and launch from within screen. This gives me the qemu console so i can wake the system up from sleep and such if needed.

I plan to reintroduce hugepages once i get the extra memory installed, although the guide here doesnt go into detail on using hugepages for 2 guests, you can simply add it to fstab from what i can tell and point each system to its relevant hugepage mount point.

Here are pictures of the final setup **(the pictures were lost in a host transition, once i move into the new house, I will take new ones)**

I've since added a usb hub on to each desk to reduce the cable clutter. Note: You do not need to passthrough the usb hub in order to use the connected usb devices.

---

**Issue1:**&nbsp;Resolved!

that all said, the second system won't use the video card. i deleted the vga none line and built the whole system in qemu and in windows, i dont even see the video card listed in any state. It seems like the pci passthrough is being completely ignored. the strange thing is, it worked once and never again.

got my second system issues resolved (shut down host, re-seat card, power up host)

---

**Issue2:**  
each system has an SSD passed through except windows detects it as a thin provisioned disk

I passed the SSD through as such:

```Shell
  -device virtio-scsi-pci,id=scsi \
  -drive file=/dev/sdc1,id=disk0,if=none,format=raw,cache=none,aio=native \
  -device scsi-hd,drive=disk0 \
```

and

```Shell
  -device virtio-scsi-pci,id=scsi \
  -drive file=/dev/sdd1,id=disk0,if=none,format=raw,cache=none,aio=native \
  -device scsi-hd,drive=disk0 \
```

for each machine respectively. I'm assuming i did something wrong here, but setting up lvm didn't interest me as I dont need snapshots or growth potential.

I actually ended up tinkering quite a bit (to the 4year old's dismay to resolve this. I ended up passing the whole disk through rather than a partition.

`-drive file=/dev/sdd,id=disk0,if=none,format=raw,cache=none,aio=native`

this did result in a noticeable performance increase during installation, but no apparent change in normal use (im sure benchmarks would show an increase), however, it didn't resolve the root issue.  

I wonder if the issue is related to how the drive is presented to windows, more work to do on this.

---

**Issue3:**&nbsp;Resolved!

My next issue (which really doesnt matter much) is MTU

the host nic is 9000MTU, the br0 is 9000MTU, until a tap device joins, then it drops to 1500. I'm not sure how to get the tap device MTU set, but i'm actively researching it.

using this guide:  
https://linuxaleph.blogspot.com/2013/01/how-to-network-jumbo-frames-to-kvm-guest.html

---

**Issue4:**  
another issue is a slight crackle in the audio, no delays.. just crackly. im passing through the usb device via

`-usb -usbdevice host:046d:0819`

I'm still researching this particular issue. It seems to be a common issue where even passing through the whole controller doesn't resolve it. Sadly, the monitors aren't hdmi, so hdmi audio isn't an option without further expense.

---

**Issue5:**  
aside from all those, i've trained my kids to put their computers to sleep when they finish, which they do… to my dismay since they can't wake the system up! i have to go into qemu and

`sendkeys ctrl 100`

a couple times and it wakes up. I checked the ovmf config to see if there was a mouse wakeup event, but i found none. Fortunately, my wife asked that I control how much time they spend on the PC more, so manually "unsleeping" their computers gives me a little more control.

---

**Issue6:**  
The passed through audio controller on the video card simply doesn't work. Windows finds a problem with it and refuses to load. I honestly haven't dug into this because I have no hdmi monitors or audio adapters, so I don't really care.. still an open issue on my end though.

---

I actually ended up exceeding my budget by about 200$ when I decided to buy the girls both some nice office chairs and 8 more gigs of ram. I upped their allocations to 6G each keeping 4G for the host. I'm probably going to tinker with hugepages soon. Considering the hardware, I feel like everything runs pretty well. There are no complaints from the girls and the windows performance score was a 6.3, which I think is pretty good considering the CPU and video cards being used.  