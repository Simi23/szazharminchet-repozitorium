---
title: Postfix Echo
description: Echo configuration for Postfix
published: true
date: 2025-03-10T14:06:05.457Z
tags: linux
editor: markdown
dateCreated: 2025-03-10T10:19:43.074Z
---

# Introduction

This guide will show you how to create an auto-responder service for Postfix which automatically replies to all mails sent to a specified address.

# Prerequisites

This guide will use `echo@dmz.worldskills.org` as the echo address. You have to make sure this mailbox exists, e.g. if you use LDAP backend you have to create this user with this email address.

# Configuration

## Master config

Edit <kbd>/etc/postfix/master.cf</kbd> and add the following line(s) at the end to add the transport.

```bash
autoreply unix - n n - - pipe
  flags=Rq user=administrator argv=/etc/postfix/scripts/autoreply.sh $sender
```

This transport created with the [pipe](https://www.postfix.org/pipe.8.html) daemon will call the `autoreply.sh` script with the sender address as an argument.

## Autoreply script

Create the folder `/etc/postfix/scripts`. Make sure the files in this directory are accessible by the user specified in the <kbd>master.cf</kbd> config file.

Create a file called <kbd>thank-you.txt</kbd>.

```
Subject: Thank you!

Thank you for reaching out.
```

Create the script <kbd>autoreply.sh</kbd>, and make sure it has execute privilege.

```bash
#!/bin/bash
/usr/sbin/sendmail -F "Echo" -f "echo@dmz.worldskills.org" $1 < /etc/postfix/scripts/thank-you.txt
```

The `-F` and `-f` options specify the sender name and address, the recipient is provided via the argument, and the txt file is piped in as the body of the email.

## Transport map

You need to create a transport map table to specify which emails go through the transport created above.

<kbd>/etc/postfix/transport</kbd>

```bash
echo@dmz.worldskills.org autoreply:
```

Now create a hash table from this file.

```bash
postmap /etc/postfix/transport
```

Lastly, edit <kbd>/etc/postfix/main.cf</kbd> and add this file as a transport map file.

```ini
transport_maps = hash:/etc/postfix/transport
```

After restarting Postfix, messages sent to **echo<span>@dmz.worldskills.o</span>rg** should get an automatic reply message.
