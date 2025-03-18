---
title: Call Transfer, Redirect and Parking
description: Call transfer and redirect configuration for Asterisk
published: true
date: 2025-03-18T14:24:22.625Z
tags: linux
editor: markdown
dateCreated: 2025-03-18T14:24:22.625Z
---

# Unconditional Call Transfer

To redirect an extension to another extension, add the following configuration to your context in <kbd>extensions.conf</kbd>.

```c
exten = 7000,1,Transfer(PJSIP/6003)
```

In this example, when an endpoint calls the extension **7000**, they will be redirected to the channel **6003**.

> Make sure `app_transfer.so` module is loaded. (It can be persistently loaded by adding `load = app_transfer.so` to <kbd>modules.conf</kbd>
{.is-info}

# Transfer based on time condition

To redirect an extension to other extensions based on the current time, create the following config:

```c
exten = 7001,1,GotoIfTime(8:00-16:00,mon-fri,*,*,?2:3)
same = 2,Transfer(PJSIP/6002)
same = 3,Transfer(PJSIP/6003)
```

In this example when the user calls the extension **7001** the following can happen:

- If it is between 8:00-16:00 on a weekday, the user will be transferred to extension **6002**
- Otherwise, the user will be transferred to **6003**

# In-call transfer

To initiate an in-call transfer, one can utilize the phone's inbuilt transfer function, or create a feature code for transfer. A blind transfer will be configured, which means that the user initiating the transfer will not contact the extension that the other user will be transferred to.

Edit <kbd>features.conf</kbd> and add:

```ini
[featuremap]
...
blindxfer = #1
```

In your <kbd>extensions.conf</kbd> file, make sure the `Dial` app used to create the call has the options to allow transfers.

```c
...
exten = _60XX,1,Dial(PJSIP/${EXTEN},10,Tt)
```

- `t` allows the called party to transfer the calling party
- `T` allows the calling party to transfer the called party

# Call Parking

When a user parks a call, the other endpoint will be placed into a park extension, which can be retrieved by any other user.

First, edit <kbd>features.conf</kbd> and add:

```ini
[featuremap]
...
parkcall = #72
```

Edit the file <kbd>res_parking.conf</kbd>

```ini
[default]
parkext => 700
parkpos => 701-709
context => parkedcalls
parkingtime => 300
findslot => next
```

This will create parking slots **701** to **709**. The parked calls will be placed into the **parkedcalls** context.

Two things will need to be done in <kbd>extensions.conf</kbd>. The **parkedcalls** context will need to be **included in the regular context** to allow users to pick up parked calls, and the **Dial** app will need to be modified to permit parking the call.

```ini
[internal]
include = parkedcalls
...
exten = _60XX,1,Dial(PJSIP/${EXTEN},10,Kk)
```

- `k` allows the called party to park the call
- `K` allows the calling party to park the call

Now, when a user enters the `#72` feature code on the keypad, the other user will be parked. The system will read out loud the parked channel extension. After that, any user can call that extension to retrieve the call.

> Note that regardless of the location of the `include` statement, it will be **always evaluated after** the extensions in the **current context**. If multiple `include` statements are used, they are evaluated in order.
{.is-warning}

# Related documentation

- [**Call Parking**](https://docs.asterisk.org/Configuration/Features/Call-Parking/) configuration
- [**Dial**](https://docs.asterisk.org/Latest_API/API_Documentation/Dialplan_Applications/Dial/) application
- [**GotoIfTime**](https://docs.asterisk.org/Latest_API/API_Documentation/Dialplan_Applications/GotoIfTime/) application
- [**Transfer**](https://docs.asterisk.org/Latest_API/API_Documentation/Dialplan_Applications/Transfer/) application