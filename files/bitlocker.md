---
title: Bitlocker
description: Bitlocker
published: true
date: 2025-06-18T14:27:32.937Z
tags: windows
editor: markdown
dateCreated: 2025-06-18T14:24:46.984Z
---

# Bitlocker
Install Bitlocker
```powershell
Enable-WindowsOptionalFeature -Online -FeatureName BitLocker, BitLocker-Utilities -All
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