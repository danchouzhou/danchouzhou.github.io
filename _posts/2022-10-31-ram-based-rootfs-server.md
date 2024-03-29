---
layout: post
title:  "Build a RAM based filesystem server"
date:   2022-10-31 14:20:20 +0800
categories: 
tags: [Debian, Linux]
---
## Background

### Advantage
- Ultra fast after boot
- No need to consider about power failure
- Great for experiment, you'll get a fresh OS after reboot
- File based maintenance, easy to copy and paste with cross platform

### Concern
- Need lots of RAM to boot (<100MB v.s. 1.5GB)
- There is no error correction if you're using a consumer RAM

## Requirement
- A physical or virtual machine
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
 - GRUB install devices: <no change>
```
```
sudo nano /etc/default/grub
```
Add following lines
```
GRUB_TERMINAL="console serial"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
GRUB_DISABLE_OS_PROBER=true
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
sudo cp local local.original
sudo cp local local.ramfs
sudo nano local.ramfs
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
/mount/init.sh
```

### List the FAT and exFAT kernel module
```
sudo cp /etc/initramfs-tools/modules /etc/initramfs-tools/modules.original
sudo cp /etc/initramfs-tools/modules /etc/initramfs-tools/modules.ramfs
ls /lib/modules/`uname -r`/kernel/fs/fat/ | cut -f1 -d '.' | sudo tee -a /etc/initramfs-tools/modules.ramfs
ls /lib/modules/`uname -r`/kernel/fs/exfat/ | cut -f1 -d '.' | sudo tee -a /etc/initramfs-tools/modules.ramfs
ls /lib/modules/`uname -r`/kernel/fs/nls/ | cut -f1 -d '.' | sudo tee -a /etc/initramfs-tools/modules.ramfs
```

### Customize fstab for RAM based rootfs
Make a backup, add another file. Set the tmpfs as / .
```
sudo cp /etc/fstab /etc/fstab.original
echo 'none /   tmpfs size=95% 0 0' | sudo tee -a /etc/fstab.ramfs
```

### Startup script
Our customize initramfs will copy start.sh from boot medium to / , systemd will handle it after whole system boot up. This allow user to add additional stuff by editing just start.sh after archive the rootfs.tar.gz
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

### Use all customize files
Copy the customize files to default filename
```
sudo cp /usr/share/initramfs-tools/scripts/local.ramfs /usr/share/initramfs-tools/scripts/local
sudo cp /etc/initramfs-tools/modules.ramfs /etc/initramfs-tools/modules
sudo cp /etc/fstab.ramfs /etc/fstab
```

### Always print the sudo lecture
```
echo 'Defaults lecture = always' | sudo tee -a /etc/sudoers.d/privacy
```

### Archive the rootfs
```
sudo tar zcvf /tmp/rootfs.tar.gz --one-file-system /
```

### Copy the archive to home directory
```
mkdir -p /bootfiles/
cp /tmp/rootfs.tar.gz /bootfiles/.
```

## Prepare the boot files
### Build the customize initramfs
```
sudo mkinitramfs -o /bootfiles/initrd.img-`uname -r`
```

### Check if fat kernel module inclued
```
lsinitramfs /bootfiles/initrd.img-`uname -r` | grep fat
```

### Copy the kernel
```
sudo cp /boot/vmlinuz-`uname -r` /bootfiles/vmlinuz-`uname -r`
```

### Make a start up script for initramfs
```
nano /bootfiles/init.sh
```
```
#!/bin/sh

echo "Copying start.sh ..."
cp /mount/start.sh .
echo "Copying rootfs.tar.gz ..."
cp /mount/rootfs.tar.gz .
umount /mount
echo "Extracting from rootfs.tar.gz ..."
tar zxvf rootfs.tar.gz
rm rootfs.tar.gz
```

### Make a start up script for systemd startup.service
```
nano /bootfiles/start.sh
```
```
#!/bin/bash

## Network ##
sed -in '/enp/d' /etc/network/interfaces

for i in $(ip link show | grep enp | cut -f2 -d' ' | sed 's/://g'); do
	echo "" >> /etc/network/interfaces
	echo "auto ${i}" >> /etc/network/interfaces
	echo "allow-hotplug ${i}" >> /etc/network/interfaces
	echo "iface ${i} inet dhcp" >> /etc/network/interfaces
	if ! ethtool ${i} | grep -sq 'Supports Wake-on: d'; then
		echo "up ethtool -s ${i} wol g" >> /etc/network/interfaces
	fi
done

systemctl restart networking.service
```

## Switch back to original files
Now the boot files are ready, so we are going to recover modified files.
```
sudo cp /usr/share/initramfs-tools/scripts/local.original /usr/share/initramfs-tools/scripts/local
sudo cp /etc/initramfs-tools/modules.original /etc/initramfs-tools/modules
sudo cp /etc/fstab.original /etc/fstab
sudo systemctl disable startup.service
sudo rm /etc/systemd/system/startup.service
sudo systemctl daemon-reload
```

## Complicated? Use the shell script version!
- [https://github.com/danchouzhou/ramfs](https://github.com/danchouzhou/ramfs)

## Create boot medium
Insert your USB thumb drive to VirtualBox, mkfs if necessary.
```
sudo fdisk /dev/sdb
sudo mkfs.exfat /dev/sdb1
```

### Mount the disk to /boot
```
sudo umount /boot
sudo mount /dev/sdb1 /boot
```

### Copy boot files
```
sudo cp /bootfiles/* /boot
```

### Install GRUB
This will install the grub bootloader and mark the disk bootable.
```
sudo grub-install /dev/sdb
```
update-grub will generate the boot menu (grub.cfg) by detecting the  content in /boot
```
sudo update-grub
```

### Check the UUID of your thumb drive
```
sudo blkid
```

### Modify the root UUID in grub.cfg
```
sudo nano /boot/grub/grub.cfg
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
- [GRUB2中文指南第二版(上）](https://wiki.ubuntu-tw.org/index.php?title=GRUB2%E4%B8%AD%E6%96%87%E6%8C%87%E5%8D%97%E7%AC%AC%E4%BA%8C%E7%89%88%28%E4%B8%8A%EF%BC%89)
