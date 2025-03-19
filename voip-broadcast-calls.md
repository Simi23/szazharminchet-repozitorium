---
title: Broadcast calls
description: Pageing, Intercom and dialing for 10-s and swapping.
published: true
date: 2025-03-19T08:40:05.762Z
tags: linux
editor: markdown
dateCreated: 2025-03-19T08:28:58.960Z
---

# Broadcast
Broadcast calls from one customer in Asterisk. Because of Asterisk there are more way to do these 
## Pageing

`/etc/asterisk/extensions.conf`
```
exten = 1500,1,Dial(PJSIP/6002&PJSIP/6003,60)
```


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
```

`/etc/asterisk/extensions.conf`
```
; 160XX: Dial people directly w/ autopickup header
exten = _160XX,1,Set(VXML_URL=intercom=true)
same = n,Set(PJSIP_HEADER(add,Call-Info)=sip:10.1.20.10\;answer-after=0)
same = n,Page(PJSIP/${EXTEN:1})
```

## Rotating calls
`/etc/asterisk/extensions.conf`
```
exten = 999,1,Dial(PJSIP/6002,10)
same = n,Dial(PJSIP/6003,10)
same = n,Goto(1)
```