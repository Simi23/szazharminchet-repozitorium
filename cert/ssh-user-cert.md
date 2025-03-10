---
title: SSH user certificate
description: Using certificates to log in to servers.
published: true
date: 2025-03-10T10:37:00.880Z
tags: linux
editor: markdown
dateCreated: 2025-03-10T10:37:00.880Z
---

# SSH user certificate

## Servet configuration
### Generate a SSH CA.
```
ssh-keygen -t rsa -b 4096 -f /etc/ssh/ssh_ca
```
This generates:
- `/etc/ssh/ssh_ca (private CA key)`
- `/etc/ssh/ssh_ca.pub (public CA key)`

### Edit configuration file
Edit `/etc/ssh/sshd_config` file:
```
...

TrustedUserCAKeys /etc/ssh/ssh_ca.pub

...
```

### Restart SSH daemon
```
systemctl restart sshd
```


## Client configuration
