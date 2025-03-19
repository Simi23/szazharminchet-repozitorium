---
title: Queues
description: Queues in Asterisk
published: true
date: 2025-03-19T09:04:38.944Z
tags: linux
editor: markdown
dateCreated: 2025-03-19T08:51:36.363Z
---

# Queues

You can configure queues in `/etc/asterisk/queues.conf` file. Here is an example what a call center would look like:

```
[name1]
musicclass = default
strategy = ringall
timeout = 30
retry = 5
wrapuptime = 10
joinempty = yes
leavewhenempty = no
ringinuse = no
member = PJSIP/6002
member = ...

[name2]
musicclass = default
strategy = ringall
timeout = 30
retry = 5
wrapuptime = 10
joinempty = yes
leavewhenempty = no
ringinuse = no
member = PJSIP/6003
member = ...
```

You can call a queue in your dialplan with the following command:
`/etc/asterisk/extensions.conf`
```
exten = 5000,1,Queue(name1)
```