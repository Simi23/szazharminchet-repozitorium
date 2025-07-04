---
title: LDAP login and automount
description: 
published: true
date: 2025-07-04T10:47:13.088Z
tags: linux
editor: markdown
dateCreated: 2025-06-06T13:33:41.709Z
---

# LDAP login and automount

## SSSD

Install neccessary services.
```bash
apt install sssd ldap-utils
```

Create `/etc/sssd/sssd.conf` file with this content:
```
[sssd]
config_file_version = 2
domains = lego.dk
services = nss,pam

[domain/lego.dk]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://HQ-DC.billund.lego.dk
cache_credentials = True
ldap_search_base = dc=lego,dc=dk
```
> Edit oweners to **root:root** and file privileges to **600**!
{.is-warning}

```bash
systemctl restart sssd
```

Enable auto home directory creation

```bash
pam-auth-update --enable mkhomedir
```

> This setup uses **StartTLS** to authenticate with the provider so make sure that you trust the root certificate for your LDAP provider! `(/usr/local/share/ca-certificates)`
{.is-warning}


## Automount homedir

Install neccessary services.
```bash
apt install libpam-mount cifs-utils
```

Get rid off the comment on this line in `/etc/security/pam_mount.conf.xml`
```xml
...
<lusersconf name=".pam_mount.conf.xml" />
...
```


Add this part to `/etc/security/pam_mount.conf.xml` to automount user folders.
```xml
...
<pam_mount>
	<debug_enable="0">
  <volume
    			options="nodev,nosuid,nofail,nobrl,cache=none,noserverino"
          uid="%(USER)"
          path="%(USER)"
          mountpoint="/home/%(USER)"
          server="HQ-DC.billund.lego.dk"
          fstype="cifs"
  />
</pam_mount>
...
```


