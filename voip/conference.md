---
title: Conference calls
description: Conference calls in Asterisk
published: true
date: 2025-03-18T14:37:23.324Z
tags: linux
editor: markdown
dateCreated: 2025-03-18T14:37:23.324Z
---

# Conference calls

Conference calling requires the **ConfBridge** module. To configure the module, edit <kbd>confbridge.conf</kbd>

```ini
[my_bridge]
type = bridge
language = en

[my_user]
type = user
music_on_hold_when_empty = yes

[my_menu]
type = menu
* = playback_and_continue(press&digits/1&confbridge-dec-list-vol-out&press&digits/2&confbridge-inc-list-vol-out&press&digits/3&>
*1 = decrease_listening_volume
1 = decrease_listening_volume
*2 = increase_listening_volume
2 = increase_listening_volume
*3 = toggle_mute
3 = toggle_mute
*0 = no_op
```

This is a basic configuration that defines a **bridge**, a **user** and a **menu profile** for the conference call.

Now edit <kbd>extensions.conf</kbd> and add the conference-related configuration.

```ini
[internal]
; 7009: Join conference
exten = 7999,1,Gosub(sub-join_conference,s,1)

; 8000: Create conference
exten = 8000,1,Gosub(sub-create_conference,s,1)

[sub-create_conference]
exten = s,1,Set(__CONFERENCENUM=${RAND(8001,8999)})
same = n,GotoIf($[${GROUP_COUNT(${CONFERENCENUM}@conference)} > 0] ? 1)
same = n,Set(GROUP(conference)=${CONFERENCENUM})
same = n,Playback(conf-youareinconfnum)
same = n,SayDigits(${CONFERENCENUM})
same = n,ConfBridge(${CONFERENCENUM},my_bridge,my_user,my_menu)

[sub-join_conference]
exten = s,1,Read(CONFERENCENUM,enter-conf-call-number,4)
same = n,GotoIf($[${GROUP_COUNT(${CONFERENCENUM}@conference)} == 0] ? 10)
same = n,ConfBridge(${CONFERENCENUM},my_bridge,my_user,my_menu)
same = 10,Playback(conf-invalid)
same = 11,Goto(1)
```
