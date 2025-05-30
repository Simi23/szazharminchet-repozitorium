---
title: SMTP/IMAP (Single server)
description: E-mail server setup on Linux with Dovecot and Postfix
published: true
date: 2025-05-30T16:35:44.714Z
tags: linux
editor: markdown
dateCreated: 2025-05-30T14:18:48.504Z
---

# Introduction

This guide will show the installation of a simple email setup. **Postfix** and **Dovecot** will be installed on the same server, and the authentication backend will be LDAP. Postfix will still use Dovecot as its authentication mechanism. Postfix will use the virtual delivery mechanism with virtual domains.

Using SQL for aliases is covered in [this](/mail/smtp-imap) guide.

# Prerequisites

Create x509 certificate for the mail server. Place the cert and the key in `/cert`.
`/cert/mail.crt` and `/cert/mail.key` will be used. Set `600` permission to the key file.

The v3 extension used for this certificate included the FQDN as the CN, and included the IP addresses of the mail server as alternative names.

> For more information on certificate setup, visit [Certification Authority](/cert/openssl) guide.
{.is-info}

# LDAP Setup

A user in LDAP looks like this:

```
# han, people, company.com
dn: uid=han,ou=people,dc=company,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Han
sn: Sol
givenName: Han
mail: han@company.com
uid: han
loginShell: /bin/bash
homeDirectory: /home/han
uidNumber: 3001
gidNumber: 3001
```

# Postfix Setup

Install the required packages:

```bash
apt install postfix postfix-ldap
```

Select *Internet Site* during install.

Set *System mail name* to the domain name (In this case, `company.com`).

## Postfix LDAP related config

Create the directory `/etc/postfix/ldap`. This contains the files which are used to query the LDAP server for information.

This directory will contain the files needed for LDAP queries. All of these files will have the following, common beginning:

```
server_host = ldap://int-srv.company.com
version = 3

bind = yes
bind_dn = cn=admin,dc=company,dc=com
bind_pw = Passw0rd

search_base = ou=people,dc=company,dc=com
scope = sub
```

This contains all basic settings for an LDAP query.

Now, create the following files with the following queries:

<kbd>/etc/postfix/ldap/virtual_mailbox_maps</kbd>

```
<common part>

query_filter = (|(uid=%s)(mail=%s))
result_attribute = uid
result_format = /mailboxes/%s/
```


<kbd>/etc/postfix/ldap/virtual_uid_maps</kbd>

```
<common part>

query_filter = (|(uid=%s)(mail=%s))
result_attribute = uidNumber
```


<kbd>/etc/postfix/ldap/virtual_gid_maps</kbd>

```
<common part>

query_filter = (|(uid=%s)(mail=%s))
result_attribute = gidNumber
```

You can test the connection with the `postmap` command:

```bash
postmap -q mark@company.com ldap:/etc/postfix/ldap/virtual_mailbox_maps
# /mailboxes/egon.lange/

postmap -q mark@company.com ldap:/etc/postfix/ldap/virtual_uid_maps
# 3001

postmap -q mark@company.com ldap:/etc/postfix/ldap/virtual_gid_maps
# 3001
```

## Postfix main.cf

Edit `/etc/postfix/main.cf`. It should look like this:

```bash
smtpd_banner = $myhostname ESMTP $mail_name (Debian/GNU)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 3.6 on
# fresh installs.
compatibility_level = 3.6

# TLS parameters
smtpd_tls_cert_file=/cert/mail.crt
smtpd_tls_key_file=/cert/mail.key
smtpd_tls_security_level=encrypt

smtp_tls_CApath=/etc/ssl/certs
smtp_tls_security_level=may
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
mydestination = localhost
relayhost =
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all

myorigin = company.com
myhostname = mail.company.com

virtual_mailbox_domains = company.com
virtual_mailbox_base = /
virtual_mailbox_maps = ldap:/etc/postfix/virtual_mailbox_maps
virtual_uid_maps = ldap:/etc/postfix/virtual_uid_maps
virtual_gid_maps = ldap:/etc/postfix/virtual_gid_maps

smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
broken_sasl_auth_clients = yes

# Logging is optional but can help A LOT with debugging
maillog_file = /var/log/postfix.log
```

## Postfix master.cf

Find these lines in `/etc/postfix/master.cf`:
```bash
# Choose one: enable submission for loopback clients only, or for any client.
#127.0.0.1:submission inet n -   y       -       -       smtpd
#submission inet n       -       y       -       -       smtpd
#  -o syslog_name=postfix/submission
#  -o smtpd_tls_security_level=encrypt
#  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_tls_auth_only=yes

...

# Choose one: enable submissions for loopback clients only, or for any client.
#127.0.0.1:submissions inet n  -       y       -       -       smtpd
#submissions     inet  n       -       y       -       -       smtpd
#  -o syslog_name=postfix/submissions
#  -o smtpd_tls_wrappermode=yes
#  -o smtpd_sasl_auth_enable=yes
```
Uncomment the like the following:
```bash
# Choose one: enable submission for loopback clients only, or for any client.
#127.0.0.1:submission inet n -   y       -       -       smtpd
submission inet n       -       y       -       -       smtpd
#  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
#  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes

...

# Choose one: enable submissions for loopback clients only, or for any client.
#127.0.0.1:submissions inet n  -       y       -       -       smtpd
submissions     inet  n       -       y       -       -       smtpd
#  -o syslog_name=postfix/submissions
  -o smtpd_tls_wrappermode=yes
#  -o smtpd_sasl_auth_enable=yes
```

## Postfix wrap-up

Restart the postfix service:

```bash
service postfix restart
```

Verify if the service is running on desired ports:

```bash
ss -tulpn | grep master

tcp   LISTEN 0      100          0.0.0.0:25        0.0.0.0:*    users:(("master",pid=2635,fd=12))
tcp   LISTEN 0      100          0.0.0.0:465       0.0.0.0:*    users:(("master",pid=2635,fd=21))
tcp   LISTEN 0      100          0.0.0.0:587       0.0.0.0:*    users:(("master",pid=2635,fd=17))
tcp   LISTEN 0      100             [::]:25           [::]:*    users:(("master",pid=2635,fd=13))
tcp   LISTEN 0      100             [::]:465          [::]:*    users:(("master",pid=2635,fd=22))
tcp   LISTEN 0      100             [::]:587          [::]:*    users:(("master",pid=2635,fd=18))
```

# Dovecot Setup
Install required packages:

```bash
apt install dovecot-core dovecot-imapd dovecot-ldap
```

Set the Maildir location in `/etc/dovecot/conf.d/10-mail.conf`:

```
mail_location = maildir:/mailboxes/%n
```

This sets the location where the mails will be stored. Create this directory and make sure it is writable:

```bash
mkdir /mailboxes
chmod 777 /mailboxes
```


Edit `/etc/dovecot/conf.d/10-ssl.conf`. The beginning should look like this:

```bash
##
## SSL settings
##

# SSL/TLS support: yes, no, required. <doc/wiki/SSL.txt>
ssl = yes

# PEM encoded X.509 SSL/TLS certificate and private key. They're opened before
# dropping root privileges, so keep the key file unreadable by anyone but
# root. Included doc/mkcert.sh can be used to easily generate self-signed
# certificate, just make sure to update the domains in dovecot-openssl.cnf
ssl_cert = </cert/mail.crt
ssl_key = </cert/mail.key
```

Edit `/etc/dovecot/conf.d/10-auth.conf`. Set the authentication mechanisms, disable (comment) the default system authentication and enable LDAP authentication (uncomment).
```bash
disable_plaintext_auth = no
auth_mechanisms = plain login

#!include auth-system.conf.ext
!include auth-ldap.conf.ext
```

Edit `/etc/dovecot/dovecot-ldap.conf.ext`:
```bash
# Connection URI to the LDAP server
uris = ldap://int-srv.company.com

# These are the admin credentials to the LDAP server
dn = cn=admin,dc=company,dc=com
dnpass = Passw0rd

auth_bind = yes
# How to search for the user trying to authenticate
auth_bind_userdn = uid=%n,ou=people,dc=company,dc=com

# Base for the search
base = ou=people,dc=company,dc=com
scope = subtree

# Attributes to return
user_attrs = homeDirectory=home,uidNumber=uid,gidNumber=gid
# Attributes to filter for
user_filter = (&(objectClass=posixAccount)(uid=%n))

# Password checking attributes
pass_attrs = uid=user,userPassword=password
# Entry filtering
pass_filter = (&(objectClass=posixAccount)(uid=%n))
```

The next file is `/etc/dovecot/10-master.conf`.

Two things will be configured:

 - Postfix will send mails to Dovecot via LMTP, this needs to be enabled.
 - Postfix was configured to authenticate through Dovecot via TCP, the listener needs to be enabled.

```bash
service auth {
  ..

  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
}
```

The above mentioned LMTP daemon needs to be enabled in `/etc/dovecot/dovecot.conf`:

```bash
protocols = imap
```

Finally, restart Dovecot:
```bash
service dovecot restart
```

# DNS Setup

Create the following DNS records.

```bash
$ORIGIN company.com.
@						IN	MX	 10					smtp.company.com.

$ORIGIN _tcp.company.com.
_imap				IN	SRV	10	0	143	imap.company.com.
_imaps			 IN	SRV	10	0	993	imap.company.com.

_submission	IN	SRV	10	0	587	smtp.company.com.
_smtps			 IN	SRV	10	0	465	smtp.company.com.
```

> Use these exact RR hostnames (`imap`/`smtp`) because clients will specifically check for them.
{.is-info}

> Make sure to set up A/AAAA records for the hosts mentioned above. (Must be A/AAAA, not CNAME)
> 
> In this example, `smtp.company.com` would be the server running Postfix, while `imap.company.com` is the one running Dovecot.
{.is-warning}

By serving these records, clients can automatically configure themselves.

# Client Setup

The mail client should automatically work with the DNS settings.

For logging in, use only the username (without domain) and password.

# Testing

 ```bash
 doveadm auth login -x service=imap <username> [password]
 ```
 
# Next Steps

Configure webmail [using Roundcube](/mail/roundcube).

# Sources

The early version of this guide was mostly based on [this](https://nrdmnn.net/resources/2-LDAP-managed-mail-server-with-Postfix-and-Dovecot-for-multiple-domains) article.