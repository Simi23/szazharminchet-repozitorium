---
title: SSH certificates
description: Using certificates to log in and identify servers.
published: true
date: 2025-03-11T14:36:43.002Z
tags: linux
editor: markdown
dateCreated: 2025-03-10T10:37:00.880Z
---

# User Certificates

## Server configuration

### Generate an SSH CA

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

# Host Certificates

## Generating certificates

**Host certificates** can be issued to hosts to **identify themselves to clients**. This way, if the client trusts the root certificate, the **user won't need to accept the host certificate** on first connection.

First, **generate the CA**.

```bash
ssh-keygen -t rsa -b 4096 -f host_ca -C host_ca
```

This generates `host_ca` and `host_ca.pub`, which are the private and public key of the certificate.

**Now, generate a keypair for a host:**

```bash
ssh-keygen -t rsa -b 4096 -N '' -f host_cert
```

This creates `host_cert` and `host_cert.pub`. The option `-N` means no passphrase. If omitted, ssh-keygen will ask for one (you can just press enter to leave it empty)

**Sign the certificate:**

```bash
ssh-keygen -s host_ca -I host.company.com -h -n host.company.com -V +52w host_cert.pub
```

What the options mean:

 - `-s` CA keyfile
 - `-I` The identity of the certificate, ideally set this to the hostname.
 - `-h` **Specifies that this is a host certificate.**
 - `-n` The (comma separated) principals which are allowed to be used with this certificate. **Set this to FQDN** if you connect to the machine via FQDN.
 - `-V +52w` One year of validity

This will create `host_cert-cert.pub`.

## Using the certificates

On the machine you issued to host certificate to, edit <kbd>/etc/ssh/sshd_config</kbd> and add the following:

```bash
# The private key file
HostKey /path/to/host_cert

# The signed certificate
HostCertificate /path/to/host_cert-cert.pub
```

---

On any machine you want to trust these certificates, copy the content of `host_ca.pub` to <kbd>~/.ssh/known_hosts</kbd> like this:

```bash
@cert-authority *.company.com ssh-rsa AAAA3N....X2gQ== host_ca
```

`*.company.com` can be a comma-separated list of hosts which the root CA will be applied for to validate the host certificate presented by the server.

Now, when connecting, the server presents its host certificate, which will be validated with the ca certificate. You can delete all other lines in your `known_hosts` file.

**You can verify the behaviour:**

```bash
ssh -vv host.company.com 2>&1 | grep "Server host certificate"
# debug1: Server host certificate: ssh-rsa-cert-v01@openssh.com SHA256:dWi6L8k3Jvf7NAtyzd9LmFuEkygWR69tZC1NaZJ3iF4, serial 0 ID "host.company.com" CA ssh-rsa SHA256:8gVhYAAW9r2BWBwh7uXsx2yHSCjY5OPo/X3erqQi6jg valid from 2020-03-17T11:49:00 to 2021-03-16T11:50:21
# debug2: Server host certificate hostname: host.company.com
```