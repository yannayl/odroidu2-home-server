---
layout: post
title:  "External Storage"
date:	16/05/2015
---

In this post I explain how to configure automatic mount of external storage device (my Western Digital external hard drive).

I am using NTFS on the external HD, so every non-computer-guy (A.K.A my parents) can
simply plug out the device and copy files to their computer.

Partitioning and formatting the hard drive is beyond the scope of this tutorial. It is highly probable that the drive is already formatted correctly to NTFS. Otherwise, one may use [gparted](http://gparted.org/) or anything else to partition and format the hard drive.

Another possible obstacle is the power supply to the hard drive. My odroid does not supply enough electriciy to the hard drive, which results in inconsistent behaviour and failed attemtps to mount (the USB is recognized correctly, the file system can not be mounted). My solution was to create an USB cable which receives it's power supply from USB power adapter, and the data is connected to the odroid's data output.
 
###  Install ntfs-3g

_ntfs-3g_ is a package used to mount the NTFS filesystem, deprecating the
ntfs kernel module (_ntfs-3g_ is using FUSE).

```
sudo pacman -Syu --noconfirm ntfs-3g
```

### Configure Automatic Mount of the HD

Create systemd's mount file for the HD.
Replace `<UUID>` to the UUID of the NTFS partition.

The UUID can be found by connecting the hard drive to the odroid and running `ls -l /dev/disk/by-uuid`.
The UUID is the one whose target was most recently recognized by the kenerl (e.g. the last blcok device output of `dmesg | tail`).

###### /etc/systemd/system/mnt-wd.mount
```
[Unit]
Description = Western Digital External Hard Drive
Requires = local-fs.target
After = local-fs.target
BindsTo = dev-disk-by\x2duuid-<UUID>.device
After = dev-disk-by\x2duuid-<UUID>.device

[Install]
WantedBy = dev-disk-by\x2duuid-<UUID>.device

[Mount]
What = UUID=<UUID>
Where = /mnt/wd
Options = noexec,nofail,umask=0
DirectoryMode = 0666
```
After saving the configuration file,enable it by executing:

```bash
systemctl enable mnt-wd.mount
```

This configuration will mount the hard drive automatically when it is connected. It also enable other configurations to take place only if the right deivce is mounted.

