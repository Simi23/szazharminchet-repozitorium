---
title: Windows User Directories
description: 
published: true
date: 2025-08-11T09:07:17.232Z
tags: windows
editor: markdown
dateCreated: 2025-08-11T09:07:17.232Z
---

# File permissions

This guide will use `C:\Homes` as its root path. Create the directory and edit its NTFS permissions:

- Go to advanced settings
- **Delete** everyting what is not **CREATOR OWNER**, or **SYSTEM**.
- **Add** Domain Users
- **Disable inheritance**
- Press **edit**
- **Show advanced permissions**
- Set **Applies to:** This folder only
- **Check in:** Create folders, Traverse folder, List folder

Share permissions:

- Give **Full control** for **Authenticated Users**

# Creating home folders

## GPO-Based creation

This method will create user directories with the name format `%username%.%userdomain%`.

To create a home directories based on a GPO, define the following rules:

- **Computer settings > Policies > Administrative Templates > System > User Profiles**
  - Set user home folder (Enabled)
    - **Path:** `\\FQDN\Sharename`
    - **Drive letter:** Choose a letter
- **User settings > Preferences > Windows Settings > Drive Maps** (optional, if you want to rename the drive map)
  - **Action:** Replace
  - **Location:** `\\FQDN\Sharename\%username%.%userdomain%`
  - **Label:** Supply a name (can use variables, like %username%)
  - **Drive Letter:** Choose a letter


## ADUC-Based creation

This method will create user directores in any format you supply.

When creating/editing the user, supply the home directory parameter. This can be done manually in ADUC in the user object's profile tab, or in PowerShell with the `New-ADUser` or `Set-ADUser` commandlets.

```powershell
New-ADUser ... -HomeDirectory "\\FQDN\Sharename\%username%"

# Or

Set-ADUser ... -HomeDirectory "\\FQDN\Sharename\%username%"
```

# Folder redirection

If your shares use only the user's name, not the domain (`%username%`), you can easily configure user folder redirection.

Create the following GPO entries:

- **User configuraton > Policies > Windows Settings > Folder redirection**
  - Do this for all folders
  - **Setting:** Basic - redirect everyone's folder to the same location
  - **Target folder location:** Create a folder for each user under the root path
  - **Root path:** `\\FQDN\Sharename`

Alternatively, if you use another folder name format, you can choose *Redirect to the following location* for the **Target folder location** setting and specify the full path, with variables.