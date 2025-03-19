---
title: Broadcast calls
description: Pageing, Intercom and dialing for 10-s and swapping.
published: true
date: 2025-03-19T08:34:53.702Z
tags: linux
editor: markdown
dateCreated: 2025-03-19T08:28:58.960Z
---

# Broadcast
Broadcast calls from one customer in Asterisk. Because of Asterisk there are more way to do these 
## Pageing
`/etc/asterisk/queues.conf`
```
...

[myqueue]
strategy = ringall
timeout = 15
retry = 5
member = PJSIP/6002
member = PJSIP/6003
```

`/etc/asterisk/extensions.conf`
```
exten = 4999,1,Answear()
same = n,Queue(myqueue)
```

## Intercom
`/etc/asterisk/extensions.conf`
```
exten = 5999,1,Page(PJSIP/6002&PJSIP/6003)
same = n,Hangup()
```

## Rotating calls
`/etc/asterisk/extensions.conf`
```
exten = 999,1,Dial(PJSIP/6002,10)
same = n,Dial(PJSIP/6003,10)
same = n,Goto(1)
```