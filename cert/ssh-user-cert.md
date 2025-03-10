---
title: SSH user certificate
description: Using certificates to log in to servers.
published: true
date: 2025-03-10T10:43:33.824Z
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
- `/etc/ssh/ssh_ca` (private CA key)
- `/etc/ssh/ssh_ca.pub` (public CA key)

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

### Generate an SSH key
```
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ha_prx01_key
```
This creates:
- `~/.ssh/ha_prx01_key` (private key)
- `~/.ssh/ha_prx01_key.pub` (public key)

###  Sign the Public Key with the SSH CA
```
ssh-keygen -s /etc/ssh/ssh_ca -I ha-prx01 -n root -V +52w ~/.ssh/ha_prx01_key.pub
```
This generates a certificate file: `~/.ssh/ha_prx01_key-cert.pub`.

### Configure SSH on the client to Use the Certificate
EditÉssh/config`