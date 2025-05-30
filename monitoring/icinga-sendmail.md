---
title: Icinga2 send email alert
description: Email alerting with postfix relay
published: true
date: 2025-05-30T14:13:06.491Z
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

Add a new user to `/etc/icinga2/conf.d/users.conf`. Make sure it has an **email** and the **enable_notification** variable.

```
object User "egon.lange" {
  import "generic-user"

  display_name = "Egon Lange"
  groups = [ "icingaadmins" ]

  enable_notifications = true

  email = "egon.lange@billund.lego.dk"
}
```

Edit the mail templates, edit the outcommented part, as you want in`/etc/icinga2/conf.d/templates.conf`.

```
...

  vars += {
    notification_icingaweb2url = "http://icinga.billund.lego.dk/icingaweb2"
    notification_from = "Icinga 2 Service Monitoring <icinga-service@billund.lego.dk>"
    notification_logtosyslog = false
  }


...
```

Add your nodes, make sure to add the **vars.notification.mail** attributes to `/etc/icinga2/conf.d/hosts.conf`!
```
object Host "MONITOR-HA-LIN-1" {
  import "generic-host"

  address = "10.100.0.10"
  address6 = "2001:db8:10:100::10"

  vars.os = "Linux"

  vars.http_vhosts["http"] = {
    http_uri = "/"
  }
  vars.disks["disk"] = {
  }
  vars.disks["disk /"] = {
    disk_partitions = "/"
  }

  vars.notification["mail"] = {
    groups = [ "icingaadmins" ]
  }
}
```

And you're done, if something goes down or up, you get an email to your mailbox!

