---
title: Icinga2 send email alert
description: Email alerting with postfix relay
published: true
date: 2025-05-30T14:08:49.806Z
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