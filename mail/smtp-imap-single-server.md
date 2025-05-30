---
title: SMTP/IMAP (Single server)
description: E-mail server setup on Linux with Dovecot and Postfix
published: true
date: 2025-05-30T14:18:48.504Z
tags: linux
editor: markdown
dateCreated: 2025-05-30T14:18:48.504Z
---

# Introduction

This guide will show the installation of a simple email setup. **Postfix** and **Dovecot** will be installed on the same server, and the authentication backend will be LDAP. Postfix will still use Dovecot as its authentication mechanism.

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

# SQL Setup

This guide will use MariaDB. Install required packages:

```bash
apt install mariadb-server
```

Set up access to the MariaDB instance following this wizard:

```bash
mysql_secure_installation
```

Log in to MariaDB as root:

```bash
mysql -u root -p
```

Run the following queries to create a database, a user, and assign permissions:

```sql
CREATE DATABASE postfix;
CREATE USER 'postfix_user'@'127.0.0.1' IDENTIFIED BY 'Passw0rd';
GRANT SELECT ON postfix.* TO 'postfix_user'@'127.0.0.1';
FLUSH PRIVILEGES;
```

Next, switch to the new database and create the schema. The `alias` field will be indexed to speed up queries.

```sql
USE postfix;
CREATE TABLE postfix_aliases (
    id INT AUTO_INCREMENT PRIMARY KEY,
    alias VARCHAR(255) NOT NULL,
    destination VARCHAR(255) NOT NULL,
    active BOOLEAN DEFAULT 1,
    INDEX (alias)
);
```

Fill the table with rows. This example creates an alias for each user with their full name, and also a group address. When you send an email to the group address, all of the returned rows will be combined, and all of the recipients will get the mail.

```sql
INSERT INTO postfix_aliases (alias, destination, active) VALUES
	('johan.bak@lego.dk', 'johan@lego.dk', 1),
	('magnus.holm@lego.dk', 'magnus@lego.dk', 1),
	('noah.klausen@lego.dk', 'noah@lego.dk', 1),
	('group@lego.dk', 'noah@lego.dk', 1),
	('group@lego.dk', 'johan@lego.dk', 1),
	('group@lego.dk', 'magnus@lego.dk', 1);
```

Exit MariaDB.

```sql
EXIT;
```

# Postfix Setup

Install the required packages:

```bash
apt install postfix postfix-ldap postfix-mysql
```

Select *Internet Site* during install.

Set *System mail name* to domain only.

## Postfix LDAP related config

Create the directory `/etc/postfix/ldap`. This contains the files which are used to query the LDAP server for information.

Create `/etc/postfix/ldap/smtpd_sender_login_maps`
```
server_host = ldap://int-srv.company.com
version = 3

bind = yes
bind_dn = cn=admin,dc=company,dc=com
bind_pw = Passw0rd

search_base = ou=people,dc=company,dc=com
scope = sub

query_filter = (|(mail=%s)(uid=%s))
result_attribute = mail
```

You can test the connection with the `postmap` command:

```bash
postmap -q mark@company.com ldap:/etc/postfix/ldap/smtpd_sender_login_maps
# mark@company.com
```

## Postfix SQL related config

Create the directory `/etc/postfix/sql`. This contains the files which are used to query the MariaDB server for information.

Create `/etc/postfix/sql/virtual_alias_maps`
```
user = postfix_user
password = Passw0rd
hosts = 127.0.0.1
dbname = postfix
query = SELECT destination FROM postfix_aliases WHERE alias='%s' AND active=1
```

You can test the connection with the `postmap` command:

```bash
postmap -q johan.bak@lego.dk ldap:/etc/postfix/sql/virtual_alias_maps
# johan@lego.dk

postmap -q group@lego.dk ldap:/etc/postfix/sql/virtual_alias_maps
# noah@lego.dk,johan@lego.dk,magnus@lego.dk
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
mydestination = localhost, localhost.localdomain
relayhost =
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 10.0.0.0/8 [2001:db8:1001::]/48
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all

myorigin = company.com
mydomain = company.com
myhostname = mail.company.com

virtual_mailbox_domains = $mydomain

# Use this line to transfer all mails to Dovecot
virtual_transport = lmtp:inet:dovecot.company.com

virtual_alias_maps = mysql:/etc/postfix/sql/virtual_alias_maps
smtpd_sender_login_maps = ldap:/etc/postfix/ldap/smtpd_sender_login_maps

smtpd_sasl_type = dovecot
smtpd_sasl_path = inet:dovecot.company.com:12345
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
apt install dovecot-core dovecot-imapd dovecot-ldap dovecot-lmtpd
```

Set the Maildir location in `/etc/dovecot/conf.d/10-mail.conf`:

```
mail_location = maildir:/mailboxes/%u
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
user_filter = (&(objectClass=posixAccount)(mail=%u))

# Password checking attributes
pass_attrs = mail=user,userPassword=password
# Entry filtering
pass_filter = (&(objectClass=posixAccount)(mail=%u))
```

The next file is `/etc/dovecot/10-master.conf`.

Two things will be configured:

 - Postfix will send mails to Dovecot via LMTP, this needs to be enabled.
 - Postfix was configured to authenticate through Dovecot via TCP, the listener needs to be enabled.

```bash
service lmtp {
  ..

  inet_listener lmtp {
    address = 10.0.0.102
    port = 24
  }
}

service auth {
  ..

  inet_listener {
    port = 12345
  }
}
```

The above mentioned LMTP daemon needs to be enabled in `/etc/dovecot/dovecot.conf`:

```bash
protocols = imap lmtp
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

Enter your email address and password.

# Testing

 ```bash
 doveadm auth login -x service=imap <username> [password]
 ```
 
# Next Steps

Configure webmail [using Roundcube](/mail/roundcube).

# Sources

The early version of this guide was mostly based on [this](https://nrdmnn.net/resources/2-LDAP-managed-mail-server-with-Postfix-and-Dovecot-for-multiple-domains) article.