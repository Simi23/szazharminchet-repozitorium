---
title: Zabbix send notification
description: How to configure notification send.
published: true
date: 2025-06-11T13:19:10.718Z
tags: linux
editor: markdown
dateCreated: 2025-06-11T13:12:14.264Z
---

# Zabbix 6.0 send notification

When you have some services and hosts you're monitoring, it's better when you get notificated if something goes to up or down state.

## Media types

> You have to go to `Administration/Media types` tab. Here you will find all preconfigured media types, and if you have created some then those too.
{.is-info}

### Email
> Look for one named **Email**, there are two from them, you can set whichever you want, the plain email will send a plain text, but the **Email (HTML)** will send a HTML formatted text through email. If you have choosen one (or you can use both), set these settings:
{.is-info}

- **SMTP Server**: Your SMTP server (or google or whatever you want to user)
- **SMTP server port**: Your SMTP servers port
- **SMTP helo**: Your domain tag
- **SMTP email**: The email that will be the sender email
- **Connection security**: Choose which you want and admint your SSL settings as needed
- **Authentication**: Username and password authentication and then fill the **Username** and **Password** field!

> CLICK UPDATE!!
{.is-warning}




### Custom script


## User medias
