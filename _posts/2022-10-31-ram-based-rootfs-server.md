---
layout: post
title:  "Build a RAM based filesystem server"
date:   2022-10-31 14:20:20 +0800
categories: 
tags: [Debian, Linux]
---
## Background
## Requirement
- At least 1G disk to store boot files
- At least 2G RAM to store rootfs, above 4G is recommend

## Enviroment
### Install Debian in VirtualBox with separate /boot partition
So that allow us to umount the /boot and mount specific disk which we want to build as boot medium.

### Config grub and linux command line
This will allow us to access the GRUB and CLI via serial console
```
sudo dpkg-reconfigure grub-pc
 - Linux command line: console=tty0 console=ttyS0,115200n8
 - Linux default command line: <empty>
 - GRUB install devices: /dev/sda
```
```
sudo nano /etc/default/grub
```
Add following lines
```
GRUB_TERMINAL="console serial"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
```
Update GRUB
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### Install additional software
```
sudo apt update && sudo apt install htop screen wget tcpdump dosfstools exfat-utils mdadm lvm2 linux-cpupower ethtool -y
sudo apt clean
```

### Customize initramfs script for RAM based rootfs
Make a backup, add another file. 
```
cd /usr/share/initramfs-tools/scripts/
sudo cp local local.origin
sudo cp local local.ram
sudo nano local.ram
```
Find local_mount_root(), comment out the normal root mount command and add following lines.
```
# if ! mount ${roflag} ${FSTYPE:+-t "${FSTYPE}"} ${ROOTFLAGS} "${ROOT}" ">
#         panic "Failed to mount ${ROOT} as root file system."
# fi
```
```
mkdir /mount
mount -t ${FSTYPE} ${ROOT} /mount
mount -t tmpfs -o size=95% none ${rootmnt}
cd ${rootmnt}
echo "Copying start.sh ..."
cp /mount/start.sh .
echo "Copying rootfs.tar.gz ..."
cp /mount/rootfs.tar.gz .
umount /mount
echo "Extracting from rootfs.tar.gz ..."
tar zxvf rootfs.tar.gz
rm rootfs.tar.gz
```

### Customize fstab for RAM based rootfs
Make a backup, add another file. Set the tmpfs as / .
```
sudo cp /etc/fstab /etc/fstab.origin
sudo sh -c "echo 'none /   tmpfs size=95% 0 1' > /etc/fstab.ram"
```

### Startup script
Our customize initramfs will copy start.sh from boot medium to / , systemd will handel it after whole system boot up. This allow user to add additional stuff by editing just start.sh after archive the rootfs.tar.gz
```
sudo nano /etc/systemd/system/startup.service
```
```
[Unit]
Description=Startup script
After=network.target

[Service]
ExecStart=/start.sh

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl daemon-reload
sudo systemctl enable startup.service
```

## Pack the rootfs
### Make a ramdisk
```
sudo mount -t tmpfs -o size=2G tmpfs /tmp
```

### Copy the initramfs script and fstab to default filename
```
# cd /usr/share/initramfs-tools/scripts/
sudo cp local.ram local
sudo cp /etc/fstab.ram /etc/fstab
```

### Archive the rootfs
```
sudo tar zcvf /tmp/rootfs.tar.gz --one-file-system /
```

### Switch back to normal
```
sudo cp local.origin local
sudo cp /etc/fstab.origin /etc/fstab
```

### Copy the archive to home directory
```
mkdir ~/bootfiles/
cp /tmp/rootfs.tar.gz ~/bootfiles/.
```

## Build the boot files
### List the fat kernel module
```
sudo cp /etc/initramfs-tools/modules /etc/initramfs-tools/modules.origin
sudo cp /etc/initramfs-tools/modules /etc/initramfs-tools/modules.ram
sudo sh -c "ls /lib/modules/5.10.0-19-amd64/kernel/fs/fat/ | cut -f1 -d '.' >> /etc/initramfs-tools/modules.ram"
sudo sh -c "ls /lib/modules/5.10.0-19-amd64/kernel/fs/nls/ | cut -f1 -d '.' >> /etc/initramfs-tools/modules.ram"
sudo cp /etc/initramfs-tools/modules.ram /etc/initramfs-tools/modules
```

### Copy the customize script
```
sudo cp local.ram local
```

### Build the customize initramfs
```
sudo mkinitramfs -o ~/bootfiles/initrd.img-`uname -r`
```

### Check if fat kernel module inclued
```
lsinitramfs ~/bootfiles/initrd.img-`uname -r` | grep fat
```

### Switch back to original files
```
sudo cp /etc/initramfs-tools/modules.origin /etc/initramfs-tools/modules
sudo cp local.origin local
```

### Copy the kernel
```
sudo cp /boot/vmlinuz-`uname -r` ~/bootfiles/vmlinuz-`uname -r`
```

## Create boot medium
Insert your USB thumb drive to VirtualBox, mkfs if necessary.
```
sudo fdisk /dev/sdb
sudo mkfs.vfat /dev/sdb1
```

### Mount the disk to /boot
```
sudo umount /boot
sudo mount /dev/sdb1 /boot
```

### Copy boot files
```
sudo cp ~/bootfiles/* /boot
```

### Install GRUB
```
sudo grub-install /dev/sdb
sudo update-grub
```

### Chenk the UUID of your thumb drive
```
sudo blkid
```

### Modify the root UUID in grub.cfg
```
sudo nano /boot/grub/grub.cfg
```

### Make a start up script
```
sudo nano /boot/start.sh
```
```
#!/bin/bash

## Network ##
dhclient -r
echo 'timeout 3' >> /etc/dhcp/dhclient.conf
echo 'retry 5' >> /etc/dhcp/dhclient.conf
for i in $(ip link show | grep enp | cut -f2 -d' ' | sed 's/://g'); do
	ip link set ${i} up
	sleep 10
	dhclient
done
```

### Unmount and poweroff the virtual machine
```
sudo umount /boot
sudo poweroff
```
Your disk is ready to boot!!

## Reference
- [Linux - Load your root partition to RAM and boot it - Tutorials - reboot.pro](http://reboot.pro/topic/14547-linux-load-your-root-partition-to-ram-and-boot-it/)
- [RAMboot How-To for Debian 8 Jessie - LinuxQuestions.org](https://www.linuxquestions.org/questions/blog/isaackuo-112178/ramboot-how-to-for-debian-8-jessie-37165/)
- [Chapter 3. The system initialization - Debian Reference](https://www.debian.org/doc/manuals/debian-reference/ch03.html)
- [systemd - Debian Wiki](https://wiki.debian.org/systemd)
