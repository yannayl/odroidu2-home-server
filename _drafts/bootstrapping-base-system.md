---
layout: post
title:  "Bootstrapping Archlinux for odroid-u2"
date:   2014-05-06 23:05:58
categories: archlinux
---

In this post, I will explain how to build a minimal Archlinux image for the odroid u2 (from scratch).
It is based on [http://forum.odroid.com/viewtopic.php?f=52&t=3662] and some other posts from the internet (see some links in the end).

1. Setting up environment
	I am building the image on x86/mint platform, but I suppose any linux distro will do (mutatis mutandis).

1.1 Working directory and environment
	{% highlight bash %}
	mkdir alarm-base-guide
	cd alarm-base-guide
	export GUIDE=`pwd`
	export SDCARD=/dev/sdX
	{% endhighlight %}
	replace the X in the SDCARD variable with the appropriate letter (e.g. /dev/sdc)
	BEWARE: specifying the wrong letter my do evil things to your storage devices.
	

1.2 Install required packages
	{% highlight bash %}
	apt-get install qemu binfmt-support qemu-user-static u-boot-tools f2fs-tools gcc-arm-linux-gnueabi
	{% endhighlight %}

1.3 Download sources and pre-build binraies
	pre-built bootloaders and their installation script
	{% highlight bash %}
	wget odroid.in/guides/ubuntu-lfs/boot.tar.gz
	{% endhighlight %}
	kernel sources and configuration files
	{% highlight bash %}
	git clone --depth 0 https://github.com/yannayl/linux-odroid.git -b odroid-3.8.y odroid-3.8.y
	{% endhighlight %}
    (you may download hardkernel's repo instead, but make sure to fix their kernel configuration. see Building the Kernel)
	script for building and chrooting the base system (that's why we need qemu)
	{% highlight bash %}
	wget https://raw.githubusercontent.com/yannayl/arch-bootstrap/master/arch-bootstrap.sh
	{% endhighlight %}

2. Building and Installing the Image

2.1 Installing the Bootloader
	The bootloader comes pre-build from hardkernel. In fact, the Exynos has couple of bootloaders, each boots the other. Thos bootloaders should be installed on the sdcard before the first partition. Inside the boot.tar.gz you will find those bootloaders alongwith a script to install them. The script is very simple, just few dd commands. Reading those commands enables one to compute where the first partition may start.

	cd $GUIDE
	tar xf boot.tar.xz
	cd boot
	chmod +x sd_fusing.sh
	sudo ./sd_fusing.sh $SDCARD

2.2 Preparing the SD-card
	Delete all partitions from the sd-card using your favorite partitioning tool.
	(example:
    {% highlight bash %}
	sudo parted $SDCARD rm 1
	sudo parted $SDCARD rm 2
    {% endhighlight %})

	Then create two partitions on the card, one for boot and one for rootfs.
	The first partition should start after at least 1240576 bytes (see Installing the Bootloader for explanation). Practically, starting it after 2M should be okay, but read first if you wanny be on the safe side.
	The boot partition should be couple of megabytes, it only contains the kernel (4M~) and boot scripts (couple of Kilobytes). Maybe in the future it will have an initrd. Practically size to 64M.
	The rootfs partition may have the rest of the sd-card.
    {% highlight bash %}
	sudo fdisk $SDCARD
	n
	p
	1
	2M
	+64M
	n
	p
	2
	66M
	<just press enter here>
	w
    {% endhighlight %})

	finally, run sudo partprobe $SDCARD to see the newly created partitions.


2.3 Make and Mount the Partitions
    {% highlight bash %}
	cd $GUIDE
	mkdir -p rootfs
	mkdir -p boot
	sudo mkfs.vfat -n boot "${SDCARD}1"
	sudo mkfs.f2fs -l rootfs "${SDCARD}2"
	mount "${SDCARD}1" boot
	mount "${SDCARD}2" rootfs
    {% endhighlight %}

	if you want an ext4 and not f2fs, don't forget to tunefs off the journal, and don't add the rootfs type in the boot cmdline.

2.4 Create Boot Script
    {% highlight bash %}
	cd $GUIDE/boot
	cat << __EOF__ | sudo tee boot.txt
	setenv initrd_high "0xffffffff"
	setenv fdt_high "0xffffffff"
	setenv bootcmd "fatload mmc 0:1 0x40008000 zImage; bootm 0x40008000"
	setenv bootargs "console=tty1 console=ttySAC1,115200n8 root=/dev/mmcblk0p2 rootwait rw mem=2047M rootfstype=f2fs init=/usr/lib/systemd/systemd"
	boot
	__EOF__
	sudo mkimage -A arm -T script -C none -n boot -d ./boot.txt boot.scr
    {% endhighlight %}

2.4 Build the Base-System
    {% highlight bash %}
	cd $GUIDE
	sudo arch-bootstrap.sh -d arch-pkg -a arm -q rootfs
    {% endhighlight %}

	Should this script fail due to network errors (if succeeds, the last line is "--done"), re-execute it. It may be usefull to use the -d switch which saves the downloaded packages.
	Note that inside the rootfs, there is a qemu-arm-static binary for chrooting from your host. You may remove it, but it is nice that you can chroot to your sd-card.

	https://wiki.archlinux.org/index.php/Archbootstrap

2.5 Configure/Build/Install the kernel
    {% highlight bash %}
	cd $GUIDE/odroid-3.8.y
	export ARCH=arm
	export CROSS_COMPILE=arm-linux-gnueabi-
	make odroidu_archlinux_defconfig
	make -j 32
	sudo cp arch/arm/boot/zImage ../boot/
	sudo make ARCH=arm INSTALL_MOD_PATH=../rootfs modules_install
    {% endhighlight %}

	Note that if you downloaded hardkernel's linux, you need to either use my defconfig or change the defconfig to support systemd, see http://wiki.gentoo.org/wiki/Systemd#Kernel for details.

2.6 Ignoring kernel updates
    {% highlight bash %}
	echo "IgnorePkg   = linux" | sudo tee -a rootfs/etc/pacman.conf
    {% endhighlight %}
	This way (hopefully) our kernel won't break

2.7 Unmount and Cleanup
	cd $GUIDE
	umount roofs boot
	sync


Thats it. You can put the sd card in the odroid and it will boot wit practically bare linux + pacman + systemd + bash + getty. 

