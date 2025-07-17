---
title: Bitlocker
description: Bitlocker
published: true
date: 2025-07-17T08:51:28.503Z
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

Enable Bitlocker to a system drive with TPM Protection
```powershell
Enable-Bitlocker -TpmProtector -MountPoint "C:"
```

Enable Bitlocker on non system drive (unlocks with C drive)
```
Enable-BitLockerAutoUnlock -MountPoint "D:"
```

