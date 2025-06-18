---
title: Bitlocker
description: Bitlocker
published: true
date: 2025-06-18T14:24:46.984Z
tags: windows
editor: markdown
dateCreated: 2025-06-18T14:24:46.984Z
---

# Bitlocker
Install Bitlocker
```powershell
Install-WindowsFeature Bitlocker -IncludeManagementTools
```

Restart the computer
```powershell
Restart-Computer
```

Enable Bitlocker to a Drive
```powershell
Enable-Bitlocker -TpmProtector -MountPoint "D:"
```