---
title: GPO
description: GPO settings for Windows
published: true
date: 2025-03-24T11:21:20.532Z
tags: windows, powershell
editor: markdown
dateCreated: 2025-03-24T08:46:54.588Z
---

# GPO
## Computer settings

### Password policies
`Policies > Windows settings > Account Policies > Password Policy`

> If you want to set the minimum password length more than 14 characters, you have to enable ***Relax minimum password length limits***.
{.is-info}


### Disable local Administrator
`Preferences > Control Panel settings > Local Users and Groups` <-- This one works

> Add Administrator with following settings
> --> **Action:** Update
> --> **User name:** Administrator
> --> **Account is disabled**
{.is-info}

`Policies > Windows Settings > Security settings > Local Policies > Security Options`

> **Accounts:** Administrator account status (disabled)
{.is-info}

### Disable CTRL+ALT+DEL
`Policies > Security Settings > Local Policies > User Rights Assignment`

> **Interactive logon:** Do not require CTRL+ALT+DEL (enabled)
{.is-info}

### Login banner
`Policies > Windows Settings > Security settings > Local Policies > Security Options`
 
> **Interactive logon:** Message text for users attempting to log on
{.is-info}


### Prevent LM hash from being stored locally in the SAM Database and Active Directory
`Policies > Windows Settings > Security settings > Local Policies > Security Options`

> **Network security:** Do not store LAN Manager hash value oin next password change
{.is-info}



### Enable users log in to DC
`Policies > Windows Settings > Security Settings > Local Policies > User Rights Assignment`

> **Allow log on locally** and define the users/groups who you want to have permission to log in to DC
{.is-info}


### Set envriotment variables
`Preferences > Windows settings > Environment Variables`

> Click add and set the variables you need!
{.is-info}


### Disable first login animation
`Policies > Administrative Templates > System > Logon`

> **Show first sign-in animation** (disabled)
{.is-info}

### Turn off file history
`Policies > Administrative Templates > Windows components > File history`

> **Turn off File History** (enabled)
{.is-info}

### Check for Windows updates
`Policies > Administrative Templates > Windows components > Windows Update`

> Set up everything you want.
{.is-info}

### Automatic mapping

#### Home directory

Set up a directory for users home directory.

NTFS Permissions for the folder:
- Go to advanced settings
- **Add** Domain Users
- Disable inheritance
- Press **edit**
- **Show advanced permissions**
- Set **Applies to:** This folder only
- **Check in:** Create folders, Traverse folder, List folder

Share permissions:
- Give **Full control** for **Authenticated Users**


`Preferences > Windows Settings > Drive Maps`

> This share will have a strange name so I recommend to change it.
> --> **Action:** Replace
> --> **Location:** \\SHARE-COMPUTER\SHARENAME\%username%.%userdomain%
> --> **Reconnect:** checked
> --> **Label as:** whatyouwant (you can use %username% .... everything you want)
> --> **Drive Letter:** **Use:** yourletter
{.is-info}


## User settings


### Automatic program start
`Policies > Windows Settings > Scripts > Logon`

> Click add and set up a new startup script.
> --> **Script name:** Define here the Path of the program.
> --> **Script parameters:** Define the parameters if you need them.
{.is-info}

### Force Wallpapers
`Policies > Administrative Templates > Desktop > Desktop`

> **Desktop Wallpaper** (create a share for domain wide setup)
{.is-info}

### Disable regedit
`Policies > Administrative Templates > System`

> --> **Don't run specified Windows applications** add *regedit.exe*
> --> **Prevent access to registry editing tools** (enabled)
{.is-info}

### Disable cmd, run and powershell
`Policies > Administrative Templates > Start Menu and Taskbar`

> **Remove Run menu from Start Menu** (enabled)
{.is-info}

`Policies > Administrative Templates > System`

> There are a little bit more settings:
> --> **Don't run specified Windows applications**: 
> --> Add *powershell.exe* and *cmd.exe*.
> --> **Prevent access to the command prompt** (enabled)
{.is-info}

### Automatic mapping

#### Default map
`Preferences > Windows Settings > Drive Maps`

> Add a new Drive with the letter you want from the network location you want. Check reconnect, and set up a name for the network drive.
{.is-info}

