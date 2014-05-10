---
layout: post
title:  "Configuring Users Account"
date:	10/05/2014
categories: account, authentication, pam
---

In this post, I explain how to configure the user accounts in the server.
How to add a user for login, disable root logins and enable local logins (from physical console) without authentication.

### Chrooting into the System ###
In the [previous post][base system] we created a chrootable Archlinux filesystem on an sdcard. Now, in order to configure the system, we need to chroot into it.
The qemu packages should be installed as instructed in the [previous post][base system].
Now, insert the sdcard to the computer, mount the rootfs, and chroot.

```bash
SDCARD=/dev/sdX
ROOTFS=/mnt/rootfs
sudo mkdir $ROOTFS
sudo mount -t f2fs "$SDCARD"2 $ROOTFS
sudo mount -o bind /proc $ROOTFS/proc # required for pacman
sudo chroot $ROOTFS
```
Replace the /dev/sdX with the sdcard device.
If the rootfs was not formatted with the flash-friendly filesystem, drop the '-t f2fs'.

### Adding a user ###
Since login as root is considered harmful and insecure, one would like to have a normal user with escalation permissions.
First, create the user, and then, set a password for this user:

```bash
USER=username
useradd -m $USER
passwd $USER
```
Enter a password for the account when prompted.

### Configuring Sudo ###
The *sudo* command is the right way to run commands with root priviliges (or root shell).
The configuration would be to grant permissions to all members of the sudo group.
Then, we add the user we created eralier to the sudo group.

```bash
pacman -S --noconfirm sudo # install the command
groupadd sudo # create sudo group
echo -e '%sudo\tALL=(ALL) ALL' >> /etc/sudoers # configure sudoers file to grant permissions to the sudo group
usermod -G sudo $USER # add username to the sudo group
```
Note: for very [good reasons][visudo doc], editing the sudoers file must be done with the *visudo* command, however, since we are chrooted from other system, and the the command is tested, I took some liberty.

### Disable root login ###
Disabling root login is considered a good practice. There are couple of ways to do so, depending on how your system is accessed. In this case, simply disabling the root account is sufficient.

```bash
passwd -l root
```

### Enable Authentication-Free Local Login ###
Warning: this is covenience configuration, which conflicts with security. I think that from physical console, one does not have to authenticate, since one can just pull out the sd-card (and nothing is encrypted). However, I could not find a good documentation for _/etc/pam.d/system-local-login_. On my current system it seems fine (only used by the *login* service), but I do not know what happens in the future.

```bash
sed -i 's,^auth,#auth,' /etc/pam.d/system-local-login
sed -i '/^#auth/a auth      sufficient pam_permit.so' /etc/pam.d/system-local-login
```

### Finish ###
Exit the chroot, unmount everything and put the sd-card in the odroid.

```bash
exit
sudo umount $ROOTFS/proc
sudo umount $ROOTFS
sudo rmdir $ROOTFS
```

[base system]: base-system.html
[visudo doc]: https://wiki.archlinux.org/index.php/sudo#Using_visudo
