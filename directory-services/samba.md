---
title: Samba
description: Samba file share configuration on Debian
published: true
date: 2025-03-10T09:30:53.698Z
tags: linux
editor: markdown
dateCreated: 2025-03-10T09:30:53.698Z
---

# Samab

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
groupadd smbshare

usermod -aG smbshare sambauser
```

### Add and enable users in samba
Add the user to samba, with the password you want, then enable it.
```
smbpasswd -a sambauser
smbpasswd -e sambauser
```

## File sharing
### Create folders

### Edit `/etc/samba/smb.conf`