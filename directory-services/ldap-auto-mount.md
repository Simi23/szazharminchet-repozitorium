---
title: LDAP login and automount
description: 
published: true
date: 2025-06-06T13:37:38.501Z
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

## Automount homedir

Install neccessary services.
```bash
apt install libpam-mount cifs-utils
```


