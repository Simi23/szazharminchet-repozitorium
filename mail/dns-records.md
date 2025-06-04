---
title: SPF, DKIM, DMARC records
description: DNS records for e-mail security and Postfix DKIM integration
published: true
date: 2025-06-04T15:01:05.272Z
tags: linux
editor: markdown
dateCreated: 2025-06-04T15:01:05.272Z
---

# SPF

An **SPF** (Sender Policy Framework) DNS record specifies the **permitted sources** of email from a certain domain.

The record itself is a **TXT** record, found at the domain root.

Example record:

```
@	IN	TXT	"v=spf1 a mx -all"
```

Meaning of the record:

- `v=spf1` identifies the record as an SPF record
- `a` means that every server in the zone having an **A** record is permitted to send mails
- `mx` means that every server in the zone having an **MX** record is permitted to send mails
- `-all` means every other IP is prohibited from sending emails from this domain

For all mechanisms, check [the wikipedia site](https://en.wikipedia.org/wiki/Sender_Policy_Framework#Mechanisms).

# DMARC

A **DMARC policy** allows a sender's domain to indicate that their email messages are protected by **SPF** and/or **DKIM**, and tells a receiver **what to do if neither of those authentication methods passes** – such as to **reject** the message or **quarantine** it. The policy can also specify how an email receiver can report back to the sender's domain about messages that pass and/or fail.

DMARC records are published in DNS with a subdomain label `_dmarc`, for example `_dmarc.example.com`.

Example record:

```
_dmarc	IN	TXT	"v=DMARC1;p=quarantine"
```

Meaning of the record:

- `v=DMARC1` identifies the record as a DMARC record
- `p=quarantine` instructs the receiving server to put the mail into quarantine if it passes neither SPF nor DKIM checks.
  Available values are *(in the order of strictness)*: `none`, `quarantine`, and `reject`.

# DKIM

**DomainKeys Identified Mail (DKIM)** is an email authentication method that permits a person, role, or organization that owns the signing domain to claim some responsibility for a message by associating the domain with the message.

The receiver can check that an email that claimed to have come from a specific domain was indeed authorized by the owner of that domain. It achieves this by affixing a **digital signature**, linked to a domain name, to each outgoing email message. The recipient system can verify this by looking up the **sender's public key published in the DNS**. A valid signature also guarantees that some parts of the email (possibly including attachments) have not been modified since the signature was affixed.

## Creating a DKIM record

To create DKIM records on Debian, first install OpenDKIM.

```bash
apt install opendkim opendkim-tools
```

Now, configure OpenDKIM by editing <kbd>/etc/opendkim.conf</kbd>.

```bash
..

Mode	sv # Uncomment this line

..

# At the sockets section, comment the first line, and uncomment the last one:
#Socket	local:/run/opendkim/opendkim.sock
#Socket	inet:8891@localhost
#Socket	inet:8891
Socket	local:/var/spool/postfix/opendkim/opendkim.sock

..

# Uncomment the InternalHosts line
#
# You can also specify your own networks here
# which should be signed instead of verified.
InternalHosts	192.168.0.0/16, 10.0.0.0/8, 172.16.0.0/12

..

# Add the following lines:
SignatureAlgorithm	rsa-sha256
KeyFile	/etc/opendkim/mail.private
```

Edit <kbd>/etc/default/opendkim</kbd> and change the **SOCKET** value to the same as before.

```bash
SOCKET="local:/var/spool/postfix/opendkim/opendkim.sock"
```

Now, configure Postfix to use OpenDKIM as a milter. Edit <kbd>/etc/postfix/main.cf</kbd>.

```
milter_protocol = 2
milter_default_action = accept
smtpd_milters = unix:/opendkim/opendkim.sock
non_smtpd_milters = unix:/opendkim/opendkim.sock
```

Create the necessary directories and set permissions. The spool directory needs to be owned by opendkim so the socket can be created. Also, the postfix user needs to be added to opendkim group so postfix can access the socket. You also need to create the directory for the keypair.

```bash
mkdir -p /var/spool/postfix/opendkim
chown opendkim:opendkim /var/spool/postfix/opendkim
usermod -aG opendkim postfix

mkdir /etc/opendkim
```

Now, generate a keypair

```bash
cd /etc/opendkim

opendkim-genkey -s mail -d lego.dk
chown opendkim:opendkim mail.private
```

- `-s` specifies the selector
- `-d` specifies the domain

Now, **add the contents** of <kbd>mail.txt</kbd> to your dns zone.

Restart both OpenDKIM and Postfix.

```bash
systemctl restart opendkim postfix
```

Great! Now you should see your emails being signed.

## Sources

Original guide: [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-dkim-with-postfix-on-debian-wheezy)