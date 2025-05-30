---
title: Icinga2 send email alert
description: Email alerting with postfix relay
published: true
date: 2025-05-30T14:10:16.591Z
tags: linux
editor: markdown
dateCreated: 2025-05-30T14:08:22.465Z
---

# Email alerting Icinga2

## Install
Install the following packages, set your servers hostname first, then your email server, to it will relay the emails.

```
apt install postfix mailutils
```

## Setting up configurations

Add a new user to `/etc/icinga2/conf.d/users.conf`. Make sure it has an email and the enable_notification variable.

```
object User "egon.lange" {
  import "generic-user"

  display_name = "Egon Lange"
  groups = [ "icingaadmins" ]

  enable_notifications = true

  email = "egon.lange@billund.lego.dk"
}

```

`/etc/icinga2/conf.d/templates.conf`

`/etc/icinga2/conf.d/hosts.conf`


