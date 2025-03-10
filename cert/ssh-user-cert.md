---
title: SSH user certificate
description: Using certificates to log in to servers.
published: true
date: 2025-03-10T11:19:40.046Z
tags: linux
editor: markdown
dateCreated: 2025-03-10T10:37:00.880Z
---

# SSH user certificate

## Server configuration
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
ssh-keygen -t rsa -b 4096 -f ~/.ssh/client_key
```
This creates:
- `~/.ssh/client_key` (private key)
- `~/.ssh/client_key.pub` (public key)

###  Sign the Public Key with the SSH CA
```
ssh-keygen -s /etc/ssh/ssh_ca -I client -n root -V +52w ~/.ssh/client_key.pub
```
This generates a certificate file: `~/.ssh/client_key-cert.pub`.

### Configure SSH on the client to Use the Certificate
Edit the `~/.ssh/config` file
```
Host ssh-server
    HostName ssh-server
    User root
    IdentityFile ~/.ssh/client_key
    CertificateFile ~/.ssh/client_key-cert.pub
```


And try login with `ssh root@ssh-server`