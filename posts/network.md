---
layout: post
title:  "Networking Configurations"
date:	10/05/2014
categories: account, authentication, pam
---

In this post, I explain how to configure static ip and run ssh server.
The first part is done in chrooted envirnment and the rest can be done over ssh.

### Chrooting into the System ###
In the [first post][base system] we created a chrootable Archlinux filesystem on an sdcard. Now, in order to configure the system, we need to chroot into it.
The qemu packages should be installed as instructed in the [first][base system].
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

### Changing the hostname ###
```bash
echo 'odroid-u2' > /etc/hostname
```

### Set Static IP  ###
Since my home router is not the most cutting edge technology, static-ip configurations must be done in the host. 

```bash
pacman -S --noconfirm netctl
cat << EOF > /etc/netctl/eth0-static
Description='A basic static ethernet connection'
Interface=eth0
Connection=ethernet
IP=static
Address=('192.168.1.50/24')
Gateway='192.168.1.1'
DNS=('8.8.8.8', '8.8.4.4')
EOF
netctl enable eth0-static
ln -s /dev/null /etc/udev/rules.d/80-net-setup-link.rules
```

Note: the last line disables udevd's new naming scheme for network interfaces. Currently, the ethernet driver of the odroid u2 does not implement name changes (at least I think it's the driver). And there is only one ethernet on the board anyway (unless one connets another one via the USB). So disabling it guarantees no sad accidents in the future.
For more info, see [udevd explanation about naming scheme](http://www.freedesktop.org/wiki/Software/systemd/PredictableNetworkInterfaceNames/).

### Avahi Daemon ###
TODO 
dhcp
`sed -i 's,#\(disallow-other-stacks\)=.*,\1=yes'`

The last line [fixes a conflict](https://forums.gentoo.org/viewtopic-p-7404160.html) between the new release of avahi and old kernels (< 3.9).

### Install & Configure SSH Server. ###

```bash
pacman -S --noconfirm openssh
systemctl enable sshd.socket
```

The default configuration is good enough. If you want more, here is a [nice article](http://www.cyberciti.biz/tips/linux-unix-bsd-openssh-server-best-practices.html).

I choose to have [systemd socket activation] (inted style) for sshd since it is not running very frequently. Configuring it to run always as normal daemon is fairly easy as well, run `systemctl enable sshd` instead of enabling the socket.

### Unchroot, Unmount, log into Odroid ###
The remaining configuration should be done on the odroid itself (due to limitations of the qemu which I do not intend to circumvent).

```bash
exit
sudo umount $ROOTFS/proc
sudo umount $ROOTFS
sudo rmdir $ROOTFS
sync
```

Remove the SD card from your host, and put it in the odroid. Connect the odroid to the network and turn it on.
Next, ssh into it:

```bash
ssh 192.168.1.50 -l username
```

Here onwards, the commands should be executed on the odroid.

### Firewall ###

```bash
sudo pacman -S --noconfirm ufw
sudo ufw allow from 192.168.1.0/24 to any app SSH
sudo ufw enable
sudo systemctl enable ufw
```

### Setting NTP and Timezone ###
This is a good place to set up NTP.
NTP is important for ssl certificate synchroziation and other stuff (curl to https breaks if the clock is not set properly).


```bash
sudo pacman -S --noconfirm ntp
sudo systemctl enable ntpd
sudo systemctl start ntpd
sudo timedatectl set-timezone Asia/Jerusalem
```
The timezones list can be found using the command `timedatectl list-timezones`.

### Bonus: Google Two Step Authenticator ###
Google Authenticator is a nice implmenetation for OTP. It has a nice mobile app
which generates a password according to the current time.
It is not really needed in a home server which is a available only from the LAN
but it is a good excuse to learn how to work with AUR.

#### Build and Install Google Authenticator ####

```bash
sudo pacman -S --noconfirm git gcc make fakeroot # requied for fetching and building
mkdir -p ~/aur-build
cd ~/aur-build
curl -o google-authenticator-libpam-git.tar.gz \
	https://aur.archlinux.org/packages/go/google-authenticator-libpam-git/google-authenticator-libpam-git.tar.gz
tar xf ./google-authenticator-libpam-git.tar.gz
cd google-authenticator-libpam-git
makepkg -s # build package
sudo pacman -U google-authenticator-libpam-git-*.pkg.tar.xz
```

#### Configure Google Authenticator for the User ####
Download Google Authenticator App to your phone. Then Execute the 
command ```google-authenticator ``` and do as requested in the dialog.
Create new account in the application either with the QR code (link generated in the dialog)
or with the codes generated in the dialog.

#### Configure SSHD and PAM ####
Now, we configure PAM to require Google Authenticator code in addition to the
default authentication (unix password). Then, we configure SSHD to add PAM ass
acceptable authentication method. Note that public-key authentication is also
enabled, and if used, no PAM method will be used at all.

```bash
sudo sed -i '/auth/a auth      required  pam_google_authenticator.so' \
	/etc/pam.d/system-remote-login
sudo sed -i 's,\(ChallengeResponseAuthentication *\)no,\1 yes,' /etc/ssh/sshd_config
sudo systemctl reload sshd
```

[base system]: base-system.html
[systemd socket activation] http://0pointer.de/blog/projects/socket-activated-containers.html
