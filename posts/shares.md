---
layout: post
title:  "Samba and NFS Shares"
date:	16/05/2015
---
### Install and Configuring Samba

```bash
sudo pacman -S samba
sudo systemctl enable smbd.socket
sudo systemctl enable nmbd
sudo ufw allow from 192.168.1.0/24 to any app CIFS
	
```

configuration files:

###### /etc/samba/smb.conf
```
[global]
	server string = %h server (Samba, Arch)
	workgroup = WORKGROUP
	map to guest = Bad Password
	syslog = 0
	log file = /var/log/samba/log.%m
	max log size = 1000
	unix extensions = yes
	guest ok = yes
	guest only = yes
	read only = yes
	include = /etc/samba/smb-shares.conf
```

###### /etc/samba/smb-rw.conf
```
[global]
	server string = %h server (Samba, Arch)
	workgroup = WORKGROUP
	map to guest = Bad Password
	syslog = 0
	log file = /var/log/samba/rw-log.%m
	max log size = 1000
	unix extensions = yes
	guest ok = yes
	guest only = yes
	writeable = yes
	include = /etc/samba/smb-shares.conf
```

###### /etc/samba/smb-shares.conf
```
[share]
	path = /mnt/wd/share
	create mask = 0777
	directory mask = 0777

[local-share]
	path = /mnt/wd/local-share
	create mask = 0777
	directory mask = 0777
```

###### /etc/systemd/system/smbd-rw.socket
```
[Unit]
Description=Samba SMB/CIFS server socket

[Socket]
ListenStream=127.0.0.1:4455
Accept=yes

[Install]
WantedBy=sockets.target
```

###### /etc/systemd/system/smbd-rw@.service
```
[Unit]
Description=Samba SMB/CIFS server instance

[Service]
ExecStart=/usr/bin/smbd -F -s /etc/samba/smb-rw.conf
ExecReload=/bin/kill -HUP $MAINPID
StandardInput=socket
```

### windows port forwarding

```
c:\cygwin64\bin\ssh.exe -TN -o UserKnownHostsFile=/cygdrive/c/Users/%USERNAME%/.ssh/known_hosts -i /cygdrive/c/Users/%USERNAME%/.ssh/id_rsa-smb -L 4455:localhost:4455 192.168.1.50
```
