---
title: LDAP login and automount
description: 
published: true
date: 2025-06-06T13:35:13.143Z
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
ldap_uri = ldap://127.0.0.1
cache_credentials = True
ldap_search_base = dc=lego,dc=dk


```


## Automount homedir

Install neccessary services.
```bash
apt install libpam-mount cifs-utils
```


