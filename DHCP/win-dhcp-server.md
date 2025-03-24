---
title: Windows DHCP server
description: Windows DHCP server configuration + PS
published: true
date: 2025-03-24T09:31:02.099Z
tags: windows, powershell
editor: markdown
dateCreated: 2025-03-24T08:41:22.712Z
---

# Windows DHCP
# Configure with PowerShell
## Installation
Install the DHCP service.
```
Install-WindowsFeature `
	-name DHCP `
  -IncludeManagementTools
```

Authorize the DHCP server if it's not standalone.
```
Add-DhcpServerInDC -DnsName NW-SRV.paris.local
```

Notify server manager, the post install is done.
```
Set-ItemProperty –Path registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\ServerManager\Roles\12 –Name ConfigurationState –Value 2
```

## IPv4 configuration
Example IPv4 configuration:
- DHCP SERVER: **NW-SRV.paris.local**
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

Create a new scope
```
Add-DhcpServerv4Scope -name "client" -StartRange 10.30.0.100 -EndRange 10.30.0.200 -SubnetMask 255.255.255.0 -State Active
```

Exclude the addresses you don't need.
```
Add-DhcpServerv4ExclusionRange -ScopeID 10.30.0.0 -StartRange 10.30.0.100 -EndRange 10.30.0.150
```

Set up the lease time:
```
Set-DhcpServerv4Scope -ScopeID 10.30.0.0 -LeaseDuration 13:13:13
```

Setting up remaining options for the DHCP scope:
```
Set-DhcpServerv4OptionValue `
    -ScopeId 10.30.0.0 `
    -DnsServer 10.10.0.10, 10.10.0.11 `
    -DnsDomain "paris.local" `
    -Router 10.30.0.1 `
    -OptionId 42 -Value "10.30.0.1" `
    -OptionId 150 -Value "10.30.0.1"
```

## IPv6 configuration
Example IPv6 configuration:
- DHCP SERVER: **NW-SRV.paris.local**
- DHCP range for **2001:db8:3010::/64**
- Default gateway: **2001:db8:3010::1**
- DNS: **2001:db8:1010::10**
- Secondary DNS: **2001:db8:1010::11**
- Lease: **2001:db8:3010::100 – 200**
- Scope name: **client**
- Exclude: **2001:db8:3010::100 – 150**
- Duration: **13 days, 13 hours, 13 minute**
- Time server (42): **2001:db8:3010::1**
- TFTP server (150): **2001:db8:3010::1**

Create a new scope
```
Add-DhcpServerv4Scope `
	-name "client" `
  -StartRange 10.30.0.100 `
  -EndRange 10.30.0.200 `
  -SubnetMask 255.255.255.0 `
  -State Active
```

Exclude the addresses you don't need.
```
Add-DhcpServerv4ExclusionRange `
	-ScopeID 10.30.0.0 `
  -StartRange 10.30.0.100 `
  -EndRange 10.30.0.150
```

Set up the lease time:
```
Set-DhcpServerv4Scope `
	-ScopeID 10.30.0.0 `
  -LeaseDuration 13:13:13
```

Setting up remaining options for the DHCP scope:
```
Set-DhcpServerv4OptionValue `
    -ScopeId 10.30.0.0 `
    -DnsServer 10.10.0.10, 10.10.0.11 `
    -DnsDomain "paris.local" `
    -Router 10.30.0.1 `
    -OptionId 42 -Value "10.30.0.1" `
    -OptionId 150 -Value "10.30.0.1"
```