---
title: Samba
description: Samba file share configuration on Debian
published: true
date: 2025-03-10T09:34:29.529Z
tags: linux
editor: markdown
dateCreated: 2025-03-10T09:30:53.698Z
---

# Samba

## Install
Install `samba` package to use samba.
```
apt install samba
```

## Users and groups
### Create users and groups local
Add a user, create a group and add this user to that group, who will have access to the share.
```
useradd -M -s /sbin/nologin sambauser
useradd -M -s /sbin/nologin jamie
groupadd smbshare

usermod -aG smbshare sambauser
```

### Add and enable users in samba
Add the user to samba, with the password you want, then enable it.
```
smbpasswd -a sambauser
smbpasswd -e sambauser
smbpasswd -a jamie
smbpasswd -e jamie
```

## File sharing
### Create folders
Create directories with the corresponding permissions.
```
mkdir -p /smb/public
chmod 2777 /smb/public
chown sambauser:smbshare /smb/public

mkdir -p /smb/private
chmod 2770 /smb/private
chown jamie:jamie /smb/public
```

### Edit `/etc/samba/smb.conf`

Add these configs to the end of the `/etc/samba/smb.conf` file
```
[public]
	path = /smb/public
	read only = yes
	guest ok = yes
	writeable = no
	force user = nobody
	force group = nogroup
	create mask = 0777
	directory mask = 0777
	write list = @smbshare

[private]
	path = /smb/private
	valid users = jamie
	guest ok = no
	writeable = yes
	create mask = 0770
	directory mask = 0770
```

Restart the service
```
systemctl restart smbd
```