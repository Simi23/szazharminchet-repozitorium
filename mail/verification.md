---
title: Email security verification
description: Verify incoming mails via SPF, DKIM and DMARC. Put non-compliant emails into Spam folder.
published: true
date: 2025-08-26T07:45:32.078Z
tags: linux
editor: markdown
dateCreated: 2025-08-05T12:17:49.904Z
---

# Introduction

This guide will show how to verify incoming emails with DKIM, SPF and DMARC checks. The guide assumes you already have these records, and DKIM signing set up. If not, visit [this guide](/mail/dns-records).

For verification to be properly tested, you need two different mail servers hosting two different mailing domains. The guide will use `hq.domain.com` and `branch.domain.com`. Each zone has an **MX** record pointing to `mail.***.domain.com`, and this is the hostname set in Postfix.

# DKIM verification

By setting up OpenDKIM, your incoming mails from other domains should already be verified. Modify <kbd>/etc/opendkim.conf</kbd>:

```c
...

# Comment out this file if you are not using DNSSEC
# TrustAnchorFile ...

# Supply your DNS server information
Nameservers 10.1.1.1

AlwaysAddARHeader yes
```

If OpenDKIM is already set up as a milter in Postfix, incoming mails will receive an **Authentication-Results** header.

# SPF and DMARC verification

To verify SPF and DMARC compliance of incoming mail, the OpenDMARC package will be used. First, install it:

```bash
apt install opendmarc
```

On the prompt asking to set up a database, select **No**. This is only needed if you plan to create aggregate DMARC reports.

**Open** <kbd>/etc/opendmarc.conf</kbd> and modify the following lines:

```c
# This should be the hostname of the mail server,
# the same that was specified in Postfix config
AuthservId mail.hq.domain.com

# Change socket to this so postfix can use it
Socket local:/var/spool/postfix/opendmarc/opendmarc.sock

# This is so OpenDMARC can trust the DKIM
# authentication result header added by OpenDKIM
TrustedAuthservIds mail.hq.domain.com

# At the end, add these two lines
IgnoreAuthenticatedClients true
SPFSelfValidate true
```

Now, **create the directory** for the socket and assign the correct permissions.

```bash
mkdir -p /var/spool/postfix/opendmarc
chmod 770 /var/spool/postfix/opendmarc
chown opendmarc:opendmarc /var/spool/postfix/opendmarc
usermod -aG opendmarc postfix
```

**Modify** <kbd>/etc/postfix/main.cf</kbd> and add the new milter.

```
smtpd_milters = unix:opendkim/opendkim.sock,unix:opendmarc/opendmarc.sock
```

# Spam folder

The config so far will **only mark noncompliant emails with headers**, but will not affect delivery of those mails. This can be achieved with a Sieve script in Dovecot, however, for that, the **delivery will need to be done by Dovecot, not Postfix**.

Postfix will transfer the mails. This assumes you are already using virtual delivery.

**Edit** <kbd>/etc/postfix/main.cf</kbd>

```c
virtual_mailbox_domains = hq.domain.com

# If Dovecot is on the same server
virtual_transport = lmtp:127.0.0.1

# If not on the same server
virtual_transport = lmtp:dovecot.hq.domain.com // Or an IP address
```

**Install** the LMTP listener for dovecot and add it as a protocol.

```bash
apt install dovecot-lmtpd
```

**Edit** <kbd>/etc/dovecot/dovecot.conf</kbd> and add this to the end:

```
protocols = imap lmtp
```

**Install** the Sieve plugin and add it as a plugin.

```bash
apt install dovecot-sieve
```

**Edit** <kbd>/etc/dovecot/conf.d/20-lmtp.conf</kbd>

```c
...

protocol lmtp {
  mail_plugins = $mail_plugins sieve
}
```

**Edit** <kbd>/etc/dovecot/conf.d/90-sieve.conf</kbd>

```c
plugin {
  ...
  # Uncomment this line
  sieve_before = /var/lib/dovecot/sieve.d/
  ...
}
```

**Create the directory** and a new file inside it, called <kbd>/var/lib/dovecot/sieve.d/spam.sieve</kbd>

```c
require ["fileinto", "regex"];

if not allof (
  header :contains "Authentication-Results" "spf=pass",
  header :contains "Authentication-Results" "dkim=pass",
  header :contains "Authentication-Results" "dmarc=pass"
) {
  fileinto "Spam";
  stop;
}
```

> When supported, you could also use negative lookaheads, like this:
> ```c
> header :regex "Authentication-Results" "dkim=(?!pass)"
> ```
> {.is-info}

**Precompile the script.**

```bash
cd /var/lib/dovecot/sieve.d
sievec spam.sieve
```

This assumes your users have a **Spam** folder. By default, it is named **Junk**. Edit <kbd>/etc/dovecot/conf.d/15-mailboxes.conf</kbd>

```c
namespace inbox {
  ...
  mailbox Spam {
    special_use = \Junk
    auto = subscribe
  }
  ...
}

```

**Restart all services.**

# Alternative Sieve script

```c
require ["fileinto", "regex"];

if anyof (
  header :contains "Authentication-Results" "spf=fail",
  header :contains "Authentication-Results" "spf=softfail",
  header :contains "Authentication-Results" "spf=temperror",
  header :contains "Authentication-Results" "spf=permerror",
  header :contains "Authentication-Results" "spf=none",
  header :contains "Authentication-Results" "dkim=fail",
  header :contains "Authentication-Results" "dkim=temperror",
  header :contains "Authentication-Results" "dkim=permerror",
  header :contains "Authentication-Results" "dkim=none",
  header :contains "Authentication-Results" "dmarc=fail",
  header :contains "Authentication-Results" "dmarc=temperror",
  header :contains "Authentication-Results" "dmarc=permerror",
  header :contains "Authentication-Results" "dmarc=none"
) {
  fileinto "Spam";
  stop;
}
```