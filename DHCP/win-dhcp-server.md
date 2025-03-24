---
title: Windows DHCP server
description: Windows DHCP server configuration + PS
published: true
date: 2025-03-24T09:20:23.588Z
tags: windows, powershell
editor: markdown
dateCreated: 2025-03-24T08:41:22.712Z
---

# Windows DHCP

Example configuration:
- DHCP range for **10.30.0.0/24**
- Default gateway: **10.30.0.1**
- DNS: **10.10.0.10**
- Secondary DNS: **10.10.0.11**
- Lease: **10.30.0.100 – 200**
- Scope name: **client**
- Exclude: **10.30.0.100 – 150**
- Duration: **13 days, 13 hours, 13 minute**
- Time server (42): **10.30.0.1**
- TFTP server (150): **10.30.0.1**

# Configure with PowerShell
## Install DHCP
```
Install-WindowsFeature `
	-name DHCP `
  -IncludeManagementTools
```