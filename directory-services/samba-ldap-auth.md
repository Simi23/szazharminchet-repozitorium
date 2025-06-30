---
title: Samba LDAP Authentication
description: Configure a Samba file server with LDAP authentication backend
published: true
date: 2025-06-30T12:43:47.295Z
tags: linux
editor: markdown
dateCreated: 2025-06-06T07:51:29.716Z
---

# Installation

Install necessary packages:

```bash
apt install samba smbldap-tools
```

# Configuration

## LDAP configuration

You need to import the Samba schema into OpenLDAP. This guide assumes you already have a running LDAP instance.

Import the schema:

```bash
ldapadd -Q -Y EXTERNAL -H ldapi:/// \
  -f /usr/share/doc/samba/examples/LDAP/samba.ldif
```

Now, the `smbldap-tools` package needs to be configured. For this to work, first set your desired **workgroup** and **netbios name** *(max 15 characters)* in <kbd>/etc/samba/smb.conf</kbd>.

After these settings are done, run the `smbldap-config` utility to configure `smbldap-tools`. Basically, all of the values you supply here should match the ones in samba configuration. When supplying LDAP suffixes, the non-root suffixes (Computers group, users group, ...) are specified relative to the main suffix, so you only need to input e.g. `ou=Computers`.

Now run the following command to populate LDAP with the necessary objects. It will automatically create all OUs specified in the previous wizard.

```bash
smbldap-populate -g 10000 -u 10000 -r 10000
```

The parameters `-g`, `-u`, and `-r` specify the starting number for **uidNumber** and **gidNumber** for the new users to be created. Select a number that doesn't overlap with existing users.

## Samba configuration

Now, edit /etc/samba/smb.conf to configure Samba to use LDAP authentication backend.

```ini
# You should have already set this up
workgroup = LEGO

# LDAP Settings
passdb backend = ldapsam:ldap://127.0.0.1
ldap suffix = dc=lego,dc=dk
ldap user suffix = ou=People
ldap group suffix = ou=Groups
ldap machine suffix = ou=Computers
ldap idmap suffix = ou=Idmap
ldap admin dn = cn=admin,dc=lego,dc=dk
ldap ssl = off # can also be 'start tls'
ldap passwd sync = yes
```

> An example configuration for these lines can be found at <kbd>/usr/share/doc/smbldap-tools/examples/smb.conf.example</kbd>
{.is-info}

Now inform Samba about the root DN password:

```bash
smbpasswd -W
```

## User authentication

The LDAP users need to show up in the system as 'Unix' users for the authentication and permissions to work properly. This will be done with SSSD.

First, install the package:

```bash
apt install sssd-ldap
```

Now, create and edit <kbd>/etc/sssd/sssd.conf</kbd>:

```ini
[sssd]
config_file_version = 2
domains = lego.dk
default_domain_suffix = lego.dk

[domain/lego.dk]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://127.0.0.1
cache_credentials = True
ldap_search_base = dc=lego,dc=dk
```

Set permissions to this file, then restart the service.

```bash
chmod 0600 /etc/sssd/sssd.conf
chown root:root /etc/sssd/sssd.conf
systemctl restart sssd
```

Restart Samba services:

```bash
systemctl restart smbd nmbd sssd
```

To test the setup, see if `getent` can list Samba groups:

```bash
getent group Replicators
```

## User management

#### Add existing LDAP users to Samba

```bash
smbpasswd -a username
```

#### Add a new user with a home directory

```bash
smbldap-useradd -a -P -m username
```

Parameters:

 - `-a` adds the Samba attributes
 - `-P` lets you enter a password for the new user
 - `-m` creates the user's home directory

#### Remove a user

```bash
smbldap-userdel username
```

#### Add a group

```bash
smbldap-groupadd -a groupname
```

#### Add a user to a group

```bash
smbldap-groupmod -m username groupname
```

#### Remove a user from a group

```bash
smbldap-groupmod -x username groupname
```
