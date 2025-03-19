---
title: IVR Menu
description: Interactive Voice Response menu configuration in Asterisk
published: true
date: 2025-03-19T08:28:30.957Z
tags: linux
editor: markdown
dateCreated: 2025-03-19T08:28:08.916Z
---

# Extension configuration

Creating an **IVR** (Internal Voice Response) menu is straight-forward, as you only need to configure your <kbd>extensions.conf</kbd> file.

For readability, the menu will get its own context, here called **menu**.

```ini
[internal]
; 000: Enter menu
exten = 000,1,Goto(menu,s,1)

[menu]
; Menu start position
exten = s,1,Background(press-1&or&press-2)
same = n,WaitExten(3)
same = n,GoTo(s,1)

exten = 1,1,Playback(you-entered)
same = n,SayNumber(1)
same = n,HangUp()

exten = 2,1,Playback(you-entered)
same = n,SayNumber(2)
same = n,HangUp()

exten = i,1,Playback(pbx-invalid)
same = n,Goto(s,1)
```

The application [**WaitExten**](https://docs.asterisk.org/Latest_API/API_Documentation/Dialplan_Applications/WaitExten/) will wait for an extension number to be entered with DTMF, then transfer the client to that extension in the current context. The [**Goto**](https://docs.asterisk.org/Latest_API/API_Documentation/Dialplan_Applications/Goto/) application is just there to restart the prompt.

In case the user dials a non-existent extension, the `i` extension will be reached which redirects them to the start of the menu.

Now, reload the dialplan, then dial extension **000** to enter the menu.

```bash
asterisk -rx "dialplan reload"
```
