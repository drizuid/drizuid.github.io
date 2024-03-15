---
title: PC for my kids update
author: Will
type: posts
date: 2018-02-01T21:27:00+00:00
slug: pc-for-my-kids-update
aliases: /2018/02/pc-for-my-kids-update/
series: ['PC for my Kids']
categories:
  - kvm
  - Linux
  - Operating Systems
  - virt
summary: I’ll start off with the bad news. After the two win10 vms were running perfectly for over a year now, I was running updates on my other linux servers… some how i did pay attention and upgraded the distro of my kvm box. It pretty much ruined everything, now I get stuck at the windows logo during boot. I even tried simply reinstalling the vm guest, but when the kvm booted from the win10 iso, it would freeze at… you guessed it, the windows logo.
---

See the original post, PC for my kids, [here](https://blog.longoconsulting.us/2017/01/pc-for-my-kids/)

I’ll start off with the bad news. After the two win10 vms were running perfectly for over a year now, I was running updates on my other linux servers… some how i did pay attention and upgraded the distro of my kvm box. It pretty much ruined everything, now I get stuck at the windows logo during boot. I even tried simply reinstalling the vm guest, but when the kvm booted from the win10 iso, it would freeze at… you guessed it, the windows logo.

I spent a little time trying to work through this, but nothing really helped. Eventually, I decided to do a full rebuild which brings us to this post.

I decided to change things up this time, I tried out ubuntu for my first time ever. Specifically the server variant. This is groundbreaking for me because I’ve always hated ubuntu for trying to be like windows, but I can’t hate on their results.. the linux userbase has grown and I think much is due to the efforts of distributions like ubuntu and linux mint. That said, I selected [Ubuntu Server 16.04.3 LTS](http://releases.ubuntu.com/16.04/ubuntu-16.04.3-server-amd64.iso.torrent).

I’m going to be fairly brief on the details here, but these are the things I did leading up to installing windows10.

Verify Support

```Shell
dmesg | grep "Virtualization Technology for Directed I/O"
[    0.646637] DMAR: Intel(R) Virtualization Technology for Directed I/O
```

Install needed packages

`apt-get install qemu-kvm qemu-utils bridge-utils iotop htop`

Find my video cards

```Shell
lspci | grep VGA
01:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Oland PRO [Radeon R7 240/340] (rev 87)
04:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Oland PRO [Radeon R7 240/340] (rev 87)
```

Get details about my video cards (2, one for each guest)

```Shell
lspci -nn | grep 01:00.
lspci -nn | grep 04:00.
01:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Oland PRO [Radeon R7 240/340] [1002:6613] (rev 87)
01:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Cape Verde/Pitcairn HDMI Audio [Radeon HD 7700/7800 Series] [1002:aab0]

04:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Oland PRO [Radeon R7 240/340] [1002:6613] (rev 87)
04:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Cape Verde/Pitcairn HDMI Audio [Radeon HD 7700/7800 Series] [1002:aab0]
```

Check the IOMMU groups

```Shell
find /sys/kernel/iommu_groups/ -type l
/sys/kernel/iommu_groups/0/devices/0000:00:00.0
/sys/kernel/iommu_groups/1/devices/0000:00:01.0
/sys/kernel/iommu_groups/1/devices/0000:01:00.0
/sys/kernel/iommu_groups/1/devices/0000:01:00.1
/sys/kernel/iommu_groups/2/devices/0000:00:14.0
/sys/kernel/iommu_groups/2/devices/0000:00:14.2
/sys/kernel/iommu_groups/3/devices/0000:00:16.0
/sys/kernel/iommu_groups/4/devices/0000:00:17.0
/sys/kernel/iommu_groups/5/devices/0000:00:1b.0
/sys/kernel/iommu_groups/6/devices/0000:00:1c.0
/sys/kernel/iommu_groups/6/devices/0000:00:1c.4
/sys/kernel/iommu_groups/6/devices/0000:04:00.0
/sys/kernel/iommu_groups/6/devices/0000:04:00.1
/sys/kernel/iommu_groups/7/devices/0000:00:1d.0
/sys/kernel/iommu_groups/8/devices/0000:00:1f.0
/sys/kernel/iommu_groups/8/devices/0000:00:1f.2
/sys/kernel/iommu_groups/8/devices/0000:00:1f.3
/sys/kernel/iommu_groups/8/devices/0000:00:1f.4
/sys/kernel/iommu_groups/9/devices/0000:00:1f.6
```

Determine if the items sharing an IOMMU group matter enough to upgrade the kernel

```Shell
lspci -nn | grep 00:01.0
00:01.0 PCI bridge [0604]: Intel Corporation Sky Lake PCIe Controller (x16) [8086:1901] (rev 07)

lspci -nn | grep 00:1c
00:1c.0 PCI bridge [0604]: Intel Corporation Sunrise Point-H PCI Express Root Port #3 [8086:a112] (rev f1)
00:1c.4 PCI bridge [0604]: Intel Corporation Sunrise Point-H PCI Express Root Port #5 [8086:a114] (rev f1)
```

Modify grub

`GRUB_CMDLINE_LINUX_DEFAULT="rd.driver.pre=vfio-pci quiet splash intel_iommu=on,igfx_off vfio_iommu_type1.allow_unsafe_interrupts=1 iommu=pt"`

and run `update-grub`

Add pci devices to /etc/modprobe.d/local.conf

`options vfio-pci ids=1002:6613,1002:aab0`

Create /etc/modprobe.d/kvm.conf and add

`options kvm ignore_msrs=1`

Add modules to initramfs via /etc/initramfs-tools/modules

```Shell
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
vhost-net
radeon
```

Blacklist the radeon driver at /etc/modprobe.d/blacklist.conf

`blacklist radeon`

run `update-initramfs -u`
Download the latest virtio drivers from [Fedora](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/virtio-win.iso)

Now we can verify some settings:

```Shell
kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used

lsmod | grep kvm
kvm_intel             172032  12
kvm                   544768  1 kvm_intel
irqbypass              16384  16 kvm,vfio_pci

lsmod | grep vfio
vfio_pci               40960  4
irqbypass              16384  16 kvm,vfio_pci
vfio_virqfd            16384  1 vfio_pci
vfio_iommu_type1       20480  2
vfio                   28672  10 vfio_iommu_type1,vfio_pci
```

I ran the typical apt-get upgrade along with installing things like screen, rsync, and vim. I removed nano and ed
The system looked like this

```Shell
No LSB modules are available.
Distributor ID: Ubuntu
Description: Ubuntu 16.04.3 LTS
Release: 16.04
Codename: xenial

Linux dznet-girls 4.4.0-87-generic #110-Ubuntu SMP Tue Jul 18 12:55:35 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
```

As you can see, the kernel is 4.4. I know many threads suggest using 4.8 to alleviate some passthrough issues, but I decided to stick with 4.4 and see how things went. If I needed to upgrade, I would at that time.  
At this point, I rebooted. I then pulled the OVMF stuff down

`wget https://sourceforge.net/projects/edk2/files/OVMF/OVMF-X64-r15214.zip/download`

Modify /etc/libvirt/qemu.conf

```Shell
nvram = [
        "/srv/ovmf-x64/OVMF_CODE-pure-efi.fd:/srv/ovmf-x64/OVMF_VARS-pure-efi.fd"
]
```

Create /etc/vfio-pci1.cfg and /etc/vfio-pci2.cfg

```
cat > /etc/vfio-pci1.cfg << EOF 
0000:01:00.0 0000:01:00.1
EOF 
cat > /etc/vfio-pci2.cfg << EOF
0000:04:00.0
0000:04:00.1
EOF
```

Setup my bridge interface /etc/network/interfaces

```Shell
# The primary network interface
auto enp0s31f6
#iface enp0s31f6 inet dhcp

auto br0
iface br0 inet dhcp
  bridge_ports enp0s31f6
  bridge_stp off
  bridge_maxwait 0
  bridge_fd 0
  post-up ip link set dev enp0s31f6 mtu 9000
```

Create /etc/udev/rules.d/70-persistent-net.rules

`SUBSYSTEM=="net", ACTION=="add", KERNEL=="tap*", ATTR{mtu}="9000"`

Create the VM launch scripts (i’ll show just 1 for brevity)

```bash
#!/bin/bash

configfile=/etc/vfio-pci1.cfg
vmname="kid1"

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

/usr/bin/qemu-system-x86_64 \
  -name $vmname,process=$vmname \
  -serial none \
  -parallel none \
  -machine type=q35,accel=kvm \
  -cpu host,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time \
  -smp 4,sockets=1,cores=2,threads=2 \
  -enable-kvm \
  -m 6G \
  -mem-prealloc \
  -balloon none \
  -rtc clock=host,base=localtime \
  -vga none \
  -nographic \
  -usb -device usb-host,vendorid=0x045e,productid=0x00db -device usb-host,vendorid=0x046d,productid=0xc01e -device usb-host,vendorid=0x046d,productid=0x0819 -device usb-host,hostbus=1,hostaddr=11 \
  -device ioh3420,bus=pcie.0,addr=1c.0,multifunction=on,port=1,chassis=1,id=root.1 \
  -device vfio-pci,host=01:00.0,bus=root.1,addr=00.0,multifunction=on,romfile=/srv/TV809MH.570 \
  -device vfio-pci,host=01:00.1,bus=pcie.0 \
  -drive if=pflash,format=raw,readonly,file=/srv/ovmf-x64/OVMF_CODE-pure-efi.fd \
  -drive if=pflash,format=raw,file=/tmp/system1.fd \
  -boot order=c \
  -object iothread,id=iothread0 \
  -device virtio-scsi-pci,id=scsi,iothread=iothread0 \
  -drive file=/dev/sdc,id=disk0,if=none,format=raw,cache=directsync,aio=native \
  -device scsi-hd,drive=disk0,bootindex=1 \
  -netdev type=tap,id=net0,ifname=tap0,vhost=on \
  -device virtio-net-pci,netdev=net0,mac=52:54:00:ea:c8:8d

#  -mem-path /dev/hugepages \
####This portion is used during installation of the guest OS####
#  -drive file=/srv/win10.iso,id=isocd,format=raw,if=none -device scsi-cd,drive=isocd,bootindex=1 \
#  -drive file=/srv/virtio-win.iso,id=virtiocd,format=raw,if=none -device ide-cd,bus=ide.1,drive=virtiocd \

exit 0
fi
```

Try to fix crackling audio

```
cat >> ~/.profile << EOF
export QEMU_AUDIO_DRV=pa
EOF
```

I then checked for the USB devices I would be passing through. Since I have only a single USB controller, I had to pass devices individually. I used the updated method of adding these (-usbdevice is deprecated in a later release of qemu.

`-usb -device usb-host,vendorid=0x04f2,productid=0x0939 -device usb-host,vendorid=0x04f2,productid=0x0833 -device usb-host,vendorid=0x045e,productid=0x075d -device usb-host,hostbus=1,hostaddr=4 \`

One of the changes I made as I set everything up was to pass through the full ssd to each system. Originally, I created a partition that I passed through, but passing through the full disk nets a noticeable performance gain.

```Shell  
   -device virtio-scsi-pci,id=scsi \
   -drive file=/dev/sdd,id=disk0,if=none,format=raw,cache=none,aio=native \
   -device scsi-hd,drive=disk0 \
```

I installed windows 10 from an ISO local to the machine on both vm guests concurrently. Install took 2minutes and 5seconds to complete.

Following the initial boot, I installed the netkvm drivers and let windows update handle the video and other missing drivers. I renamed the systems and joined them to the domain. I shut the system down fully and removed the virtio and win10 iso configs from my launch script. I booted both systems and experienced a BSOD as windows prepared the logon screen. This is actually a common issue with AMD video cards and there are some work-arounds. I didn’t use any of them, I simply rebooted again and it booted fine.  
My WEI was 6.8 on both systems and i ran `winsat formal` concurrently on them.

I used to have `kvm_intel.emulate_invalid_guest_state=0`, but i couldn’t remember why.. so now I don’t.

I don’t know if this will help anyone, but my kids game/homework on these guest vms daily (for over a year now) and the biggest complaint was the audio crackle. With that gone, a 6.8 score.. they’re pretty happy. There’s also enough resources left that I run my backup DNS (bind9) on the host.
