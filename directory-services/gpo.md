---
title: GPO
description: GPO settings for Windows
published: true
date: 2025-03-24T10:26:27.932Z
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


### Login banner
`Policies > Windows Settings > Security settings > Local Policies > Security Options`
 
> **Interactive logon:** Message text for users attempting to log on
{.is-info}

### Prevent LM hash from being stored locally in the SAM Database and Active Directory
`Policies > Windows Settings > Security settings > Local Policies > Security Options`

> **Network security:** Do not store LAN Manager hash value oin next password change
{.is-info}

### Enable users log in to DC
`Policies > Security Settings > Local Policies > User Rights Assignment`

> **Allow log on locally** and define the users/groups who you want to have permission to log in to DC
{.is-info}

### Check for Windows updates
`Policies > Administrative Templates > Windows components > Windows Update`


### Set envriotment variables
`Preferences > Windows settings > Environment Variables





### Automatic program start

### Disable regedit

### Disable "cmd" "run" and "powershell"

### Turn off file history

### Automatically map home folder

### Automatically map folders

### Wallpaper

### Disable first login animation

### Disable CTRL+ALT+DEL



## User settings


