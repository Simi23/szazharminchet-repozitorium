---
title: OpenLDAP
description: Directory services with OpenLDAP
published: true
date: 2025-03-11T14:06:07.850Z
tags: linux
editor: markdown
dateCreated: 2025-02-18T15:05:31.500Z
---

# Information

Good OpenLDAP reference can be found at [https://ldap.com/ldap-oid-reference-guide/](https://ldap.com/ldap-oid-reference-guide/).

This guide will use `lego.dk` as its base domain, so `dc=lego,dc=dk` in LDAP.

# Installation

Install the required packages.

```bash
apt install slapd ldap-utils
```

Reconfigure the package to set the base domain.

```bash
dpkg-reconfigure slapd
```

# Basic operations

## Commands

You can use a few commands to interact with the LDAP database:

- `ldapsearch`: **search** for content in the database
- `ldapadd`: **add new objects** into the database
- `ldapmodify`: **modify existing objects**
- `ldapdelete`: **delete existing objects**
- `ldappasswd`: **set password** for user objects

## Basic bindings

When you need to authenticate to use a command, you make a binding to the LDAP database. There are two base directories you will usually access:

- `dc=lego,dc=dk`, (or the domain you set) which contains all of the **LDAP objects** you create. 
- `cn=config`, which contains the **settings** of the **LDAP service** itself.

To access the **main LDAP database**, you will usually use the following options:

- `-x` to use simple authentication
- `-W` to prompt for password (you can also use `-w <password>`)
- `-D cn=admin,dc=lego,dc=dk` to specify the **binddn** to be the default admin user

To access the **configuration database**, you need the **root user**, so you need to connect to LDAP using a **UNIX socket**, with **SASL authentication**:

- `-Q` enable SASL quiet mode
- `-Y EXTERNAL` specify SASL mechanism
- `-H ldapi:///` specify LDAP URL, `ldapi` schema means UNIX socket

**Other common options:**

- `-b <base_dn>` set a **base DN** for a query
- `-f <file>` input an LDIF file
- `-Z[Z]` tries to start connection with **StartTLS** (`-ZZ` requires this to be successful)
- `-L[L[L]]` more `L`-s prints less unneeded information, e.g. comments (this applies only to `ldapsearch`)

## Modifying entries

Adding or removing objects is quite trivial, but modifying them has a few options, as you can **add**, **remove** or **modify** **parameters** of an LDAP object.

To do so, you can specify all modifications in an LDIF file. You must specify the **object DN** as always. If you need to make multiple modifications on the same DN, you can separate them using `-`, as demonstrated below. The second line is `changetype`, followed by the action, which can be `add`, `delete` or `replace`. **This line specifies which property is modified.**

An example:

```python
dn: uid=john,ou=Users,dc=lego,dc=dk
changetype: modify
replace: description
description: New description
-
add: mail
mail: john@lego.dk
-
delete: homeDirectory
```

This modifies the `uid=john` object by replacing the `description`, adding a `mail` field and deleting the `homeDirectory` field.

# Populate with data

Example directory tree:

- **Users** (OU)
  - **Noah** (User)
  - **Johan** (User)
  - **Magnus** (User)
- **Groups** (OU)
  - **Factory** (Group)

The following **LDIF** would be used:

```python
# Users OU
dn: ou=Users,dc=lego,dc=dk
objectClass: organizationalUnit
ou: Users

# Groups OU
dn: ou=Groups,dc=lego,dc=dk
objectClass: organizationalUnit
ou: Groups

# Factory group
dn: cn=factory,ou=Groups,dc=lego,dc=dk
objectClass: posixGroup
cn: factory
gidNumber: 5001
memberUid: noah
memberUid: johan
memberUid: magnus

# Noah
dn: uid=noah,ou=Users,dc=lego,dc=dk
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: noah
sn: Klausen
givenName: Noah
cn: Noah Klausen
gecos: Noah Klausen
uidNumber: 10001
gidNumber: 5001
userPassword: {CRYPT}x # This is a blank password
loginShell: /bin/bash
homeDirectory: /home/noah

# Johan
dn: uid=johan,ou=Users,dc=lego,dc=dk
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: johan
sn: Bak
givenName: Johan
cn: Johan Bak
gecos: Johan Bak
uidNumber: 10002
gidNumber: 5001
userPassword: {CRYPT}x
loginShell: /bin/bash
homeDirectory: /home/johan

# Magnus
dn: uid=magnus,ou=Users,dc=lego,dc=dk
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: magnus
sn: Holm
givenName: Magnus
cn: Magnus Holm
gecos: Magnus Holm
uidNumber: 10003
gidNumber: 5001
userPassword: {CRYPT}x
loginShell: /bin/bash
homeDirectory: /home/magnus
```

## Setting passwords

After creating users, you can set their password by running:

```bash
ldappasswd -x -W -D <ADMIN_DN> -S <USER_DN>
```

This will change the password of `USER_DN`.

---

Alternatively, you can **pregenerate** the password hash.

```bash
slappasswd -s Passw0rd
# {SSHA}...........................
```

You can insert this into the `userPassword` property when creating the user to set the password upon creation.

# TLS

LDAP can use TLS to make its connections secure. To use TLS, you need certificates. You can generate certificates using [this guide](/cert/openssl). (Make sure to include `extendedKeyUsage = serverAuth`.)

Let's assume you now have the following files:

- `/cert/ca.pem` the CA certificate
- `/cert/cert.pem` the server certificate
- `/cert/key.pem` the server keyfile

Create the file `certinfo.ldif`.

```python
dn: cn=config
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /cert/ca.pem
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /cert/cert.pem
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /cert/key.pem
```

Use the `ldapmodify` command to modify the database:

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f certinfo.ldif
```

Edit `/etc/default/slapd` and add `ldaps:///` to this line:

```
SLAPD_SERVICES="ldap:/// ldapi:/// ldaps:///"
```

Finally, restart `slapd`.

```bash
service slapd restart
```

Test StartTLS:
```bash
ldapwhoami -x -ZZ -H ldap://ldap1.lego.dk
```

Test LDAPS:
```
ldapwhoami -x -H ldaps://ldap1.lego.dk
```

# Replication

## Provider configuration

You need a user for the replication. Create `replicator.ldif`.

```python
dn: cn=replicator,dc=lego,dc=dk
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: replicator
description: Replication user
userPassword: {CRYPT}x
```

Add the file, then set a password for the replicator user.

```bash
ldapadd -x -ZZ -H ldap://ldap1.lego.dk -D cn=admin,dc=lego,dc=dk -W -f replicator.ldif
ldappasswd -x -ZZ -H ldap://ldap1.lego.dk -D cn=admin,dc=lego,dc=dk -W -S cn=replicator,dc=lego,dc=dk
```

ACLs need to be applied to the replicator user. Create `replicator-acl-limits.ldif`.

```python
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to *
  by dn.exact="cn=replicator,dc=lego,dc=dk" read
  by * break
-
add: olcLimits
olcLimits: dn.exact="cn=replicator,dc=lego,dc=dk"
  time.soft=unlimited time.hard=unlimited
  size.soft=unlimited size.hard=unlimited
```

Add it to the database:

```bash
ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f replicator-acl-limits.ldif
```

Create the replication entry, `provider_sync.ldif`.

```python
# Add indexes to the frontend db.
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcDbIndex
olcDbIndex: entryCSN eq
-
add: olcDbIndex
olcDbIndex: entryUUID eq

#Load the syncprov module.
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: syncprov

# syncrepl Provider for primary db
dn: olcOverlay=syncprov,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
olcSpCheckpoint: 100 10
olcSpSessionLog: 100
```

Add the new content:

```bash
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f provider_sync.ldif
```

## Consumer configuration

Create the file `consumer_sync.ldif`.

```python
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: syncprov

dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcDbIndex
olcDbIndex: entryUUID eq
-
add: olcSyncrepl
olcSyncrepl: rid=0
  provider=ldap://ldap1.lego.dk
  bindmethod=simple
  binddn="cn=replicator,dc=lego,dc=dk" credentials=<secret>
  searchbase="dc=lego,dc=dk"
  schemachecking=on
  type=refreshAndPersist retry="60 +"
  starttls=critical tls_reqcert=demand
-
add: olcUpdateRef
olcUpdateRef: ldap://ldap1.lego.dk
```

Add the new config:

```bash
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f consumer_sync.ldif
```

## Testing

You can monitor the replication with the following command:

```bash
ldapsearch -z1 -LLL -x -s base -b dc=lego,dc=dk contextCSN
```

If this value matches up, the servers are in sync.