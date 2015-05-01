---
layout: post
title:  "Bootstrapping Archlinux for odroid-u2"
date:   2014-05-06 23:05:58
categories: archlinux
---

In this post, I explain how to build a minimal Archlinux image for the odroid u2 (from scratch).
It is based on this [odroid forum post](http://forum.odroid.com/viewtopic.php?f=52&t=3662) and some other posts from the internet (see some links inline).

### Setting up environment

I am building the image on x86/mint platform, but I suppose any linux distro will do (mutatis mutandis).

#### Working directory and environment variables

```bash
mkdir alarm-base-guide
cd alarm-base-guide
export GUIDE=`pwd`
export SDCARD=/dev/sdX
```

replace the X in the SDCARD variable with the appropriate letter (e.g. /dev/sdc)
BEWARE: specifying the wrong letter may do evil things to your storage devices.
	

#### Install required packages

```bash
sudo apt-get install qemu binfmt-support qemu-user-static # chrooting packages
sudo apt-get install u-boot-tools # compiling the boot script
sudo apt-get install gcc-arm-linux-gnueabi # arm toolchain
sudo apt-get install f2fs-tools # making the falsh-friendly file-system
```
If ext4 is preferred over the f2fs, install the ext4 package instead.

#### Download sources and pre-build binraies

```bash
wget odroid.in/guides/ubuntu-lfs/boot.tar.gz # boot image and installation script
git clone --depth 1 https://github.com/yannayl/linux-odroid.git -b odroid-3.8.y odroid-3.8.y # kernel sources and configuration files
wget https://raw.githubusercontent.com/yannayl/arch-bootstrap/master/arch-bootstrap.sh # script for building and chrooting the base system
```

([hardkernel's repo](https://github.com/hardkernel/linux) may be downloaded instead, but make sure to fix their kernel configuration. see Building the Kernel section)

### Building and Installing the Image

#### Installing the Bootloader

The bootloader comes pre-built from hardkernel. In fact, the Exynos has couple of bootloaders, each boots the other. Thos bootloaders should be installed on the sdcard before the first partition. Inside the boot.tar.gz you will find those bootloaders alongwith a script to install them. The script is very simple, just few dd commands. Reading those commands enables one to compute where the first partition may start.

```bash
cd $GUIDE
tar xf boot.tar.gz
cd boot
chmod +x sd_fusing.sh
sudo ./sd_fusing.sh $SDCARD
```

#### Preparing the SD-card

Delete all partitions from the sd-card using your favorite partitioning tool.
(example:
```
sudo parted $SDCARD rm 1;
sudo parted $SDCARD rm 2
```
)

Then create two partitions on the card, one for boot and one for rootfs.
The first partition should start after at least 1240576 bytes (see Installing the Bootloader section or explanation). Practically, starting it after 1.5M should be okay, but read first if you wanna be on the safe side.
The boot partition should be couple of megabytes, it only contains the kernel (4M~) and boot scripts (couple of Kilobytes). Maybe in the future it will have an initrd. Practically size to 64M.
The rootfs partition may have the rest of the sd-card.
The following numbers are under the assumption of 512 bytes sector. One may verify it is the case executing the 'p' command in fdisk.

```bash
sudo fdisk $SDCARD
p
n
p
1
4096
+131072
n
p
2
135169
<just press enter here>
w
```

finally, run ```sudo partprobe $SDCARD``` to see the newly created partitions.


#### Make and Mount the Partitions

```bash
cd $GUIDE
mkdir -p rootfs
mkdir -p boot
sudo mkfs.vfat -n boot "${SDCARD}1"
sudo mkfs.f2fs -l rootfs "${SDCARD}2"
sudo mount "${SDCARD}1" boot
sudo mount -t f2fs "${SDCARD}2" rootfs
```

if you want an ext4 and not f2fs, don't forget to tunefs off the journal, and don't add the rootfs type in the boot cmdline.

#### Create Boot Script

```bash
cd $GUIDE/boot
cat << __EOF__ | sudo tee boot.txt
setenv initrd_high "0xffffffff"
setenv fdt_high "0xffffffff"
setenv bootcmd "fatload mmc 0:1 0x40008000 zImage; bootm 0x40008000"
setenv bootargs "console=tty1 console=ttySAC1,115200n8 root=/dev/mmcblk0p2 rootwait rw mem=2047M rootfstype=f2fs init=/usr/lib/systemd/systemd"
boot
__EOF__
sudo mkimage -A arm -T script -C none -n boot -d ./boot.txt boot.scr
```

#### Build the Base-System

```bash
cd $GUIDE
sudo ./arch-bootstrap.sh -d arch-pkg -a arm -q rootfs
```

Should this script fail due to network errors (if succeeds, the last line is "--done"), re-execute it. It may be usefull to use the -d switch which saves the downloaded packages.
Note that inside the rootfs, there is a qemu-arm-static binary for chrooting from the host. You may remove it, but it is nice that you can chroot to your sd-card.

I altered the original script which was documented (sorta) here:[Archbootstrap in ArchLinux Wiki](https://wiki.archlinux.org/index.php/Archbootstrap)

#### Configure/Build/Install the Kernel

```bash
cd $GUIDE/odroid-3.8.y
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabi-
make odroidu_archlinux_defconfig
make -j 32
sudo cp arch/arm/boot/zImage $GUIDE/boot/
sudo make ARCH=arm INSTALL_MOD_PATH=$GUIDE/rootfs modules_install
```

Note that if you downloaded hardkernel's linux, you need to either use my defconfig or change the defconfig to support systemd, as described in [systemd's readme](http://cgit.freedesktop.org/systemd/systemd/tree/README#n44).
If f2fs used as root filesystem, don't forget to enable it as well.

* Ignoring kernel updates

```bash
sudo sed -i '/^#IgnorePkg/a IgnorePkg   = linux' $GUIDE/rootfs/etc/pacman.conf
```
This way (hopefully) our kernel won't break

#### Unmount and Cleanup

```bash
cd $GUIDE
sudo umount roofts boot
sync
```

### Finish 

Thats it. You can put the sd card in the odroid and it will boot wit practically bare linux + pacman + systemd + bash + getty. 

