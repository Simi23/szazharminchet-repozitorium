---
title: Samba custom homes directory
description: Set a custom path for Samba user home directories with auto creation on first logon
published: true
date: 2025-06-06T08:10:39.309Z
tags: linux
editor: markdown
dateCreated: 2025-06-06T08:10:39.309Z
---

# Configuration

To set a custom path for the user's home directories and automatically create that directory on first logon, extend the `[homes]` section in your Samba configuration file, <kbd>/etc/samba/smb.conf</kbd> to look like the following:

```ini
[homes]
   comment = Home directories
   browseable = no
   
   # Change this to no to allow users to write files
   read only = no
   
   create mask = 0700
   directory mask = 0700
   valid users = %S
   
   # These are the new config lines
   path = /share/users/%S
   root preexec = bash -c 'mkdir -p /share/users/%S; chown %U:nogroup /share/users/%U; chmod 700 /share/users/%U'
```

After configuration, restart Samba service:

```bash
systemctl restart smbd
```