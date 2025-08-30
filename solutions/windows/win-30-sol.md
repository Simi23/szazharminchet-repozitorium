---
title: ES25 - ModB - 30% Solution
description: 
published: true
date: 2025-08-30T18:16:06.215Z
tags: windows, es25-windows
editor: markdown
dateCreated: 2025-08-19T11:19:07.140Z
---

# Changes

## NTP

DC, GPO: `Computer > Policies > ADMX > System > Windows Time Service > Time Providers`

  - **Configure Windows NTP Client:** Enabled
    - **NtpServer:** `<FQDN>,0x8`
    - **Type:** NTP
  - **Enable Windows NTP Client:** Enabled
  - **Enable Windows NTP Server:** Enabled

## BGP

Powershell commandlets:

```powershell
Add-BgpRouter `
  -BgpIdentifier 1.1.1.1 `
  -LocalASN 65111 `
  -IPv6Routing Enabled `
  -LocalIPv6Address 2001:100::1 `
  -TransitRouting Enabled

Add-BgpPeer `
	-Name HelloWorld `
  -LocalIPAddress 1.1.1.1 `
  -PeerIPAddress 2.2.2.2 `
  -PeerASN 65222

Add-BgpCustomRoute `
  [-Interface ..] `
  -Network 1.1.1.0/24
```

## NLB

**Mode:** multicast

## NIC Teaming

**Load balancing:** Address hash
**Standby:** 3 interfaces

## RDS

"Next next" install (Gergő megoldja)

## Access-Denied Message

GPO: `Computer > Policies > ADMX > System > Access-Denied Assistance`

  - Customize message for access denied errors: Enabled

## Remote client

Use same \_vpn certificate template. Import certificate for localmachine.

On the client, add an IKEv2 VPN, in more settings select machine certificates in the Security tab.

On the server, enable **IPv4** and **IPv6** remote access. Enable RAS on the IKEv2 ports. Set DNS address on all local interfaces so the client will definitely receive it.

## Disaster plan

Gergő megoldja

## Helpdesk

```powershell
New-ADGroup `
  -Name Helpdesk `
  -Scope Global `
  -Path "OU=Groups,OU=Skills,DC=skillsnet,DC=dk"
```

Right click on *Employees* OU, open *Delegate Control...* wizard. Select the *Helpdesk* group, and check *Reset password...*.

## Print server

**SRV2:**

```powershell
Install-WindowsFeature Print-Server
```

**DC:**

```powershell
Install-WindowsFeature RSAT-Print-Services
```

**GPO:** `User Config > Preferences > Control Panel > Printers`

  - Add shared printer

## DNSSEC

Create the following zones on **INET**:

- . (root)
- dk
- net

Sign all zones on **INET** and **DC** using the **DNSSEC > Sign zone** wizard. Go with the default settings.

Alternatively, a zone can be signed with the command:

```powershell
Invoke-DnsServerZoneSign -ZoneName "skillsdev.dk" -SignWithDefault
```

After all zones are signed, each zone's **DS records** must be imported into the respective **parent zone**. These records are automatically put into a file with the following name syntax:

```
C:\Windows\System32\dns\dsset-<zone_name>
```

> For the root zone, *zone_name* is an empty string.
{.is-info}

Import each file into the parent zone. *(For some zones you will need to copy files using scp.)*

```powershell
# This will import the DS records for the dk zone into the root zone
Import-DnsServerResourceRecordDS `
  -ZoneName . `
  -DSSetFile C:\Windows\System32\dns\dsset-dk
```

After the whole chain is created, you need to **import** the **keyset of the root zone** into any resolvers where you want the chain to be trusted *(in this case, **DC** and **DEV-SRV**)*. 

This can be done with the **Import DNSKEY** wizard in the **Trust Points** section, or via the command:

```powershell
Import-DnsServerTrustAnchor -KeySetFile "C:\Windows\System32\dns\keyset-"
```

## DNS Backup

```powershell
function DNS-Backup {
  # Variables
  $i = 1
  $dnsServers = @("DC.skillsnet.dk", "SRV.skillsdev.dk", "INET.skillspublic.dk")
  $password = ConvertTo-SecureString "Passw0rd!" -AsPlainText -Force
  $backupPath = "C:\Resources\DNS"

  # Logic
  Backup-Start("DNS zones")
  Directory-Create($backupPath)

  foreach ($dnsServer in $dnsServers) {
      $parts = $dnsServer.Split('.')
      $domain = ($parts[1..$part.Length] -join '.')
      $localPath = Join-Path $backupPath -ChildPath $domain

      Write-Host "Backing up '$($domain)' DNS zone"

      ssh Administrator@dnsServer "if (Test-Path 'C:\Windows\System32\dns\zone') { Remove-Item 'C:\Windows\System32\dns\zone' } Export-DnsServerZone -Name $($domain) -FileName zone"
      scp "Administartor@$($dnsServer):C:\Windows\System32\dns\zone" $localPath

      $i++
      Write-Host "$($domain) zone backup done!" -ForegroundColor Green
  }
  Write-Host "DONE with DNS backup!" -ForegroundColor Green
}


# Variables
$backupRoot = "C:\Backups"

# Running functions
try {
  Directory-Create($backupRoot)
  DNS-Backup
  $subject = "Backup success"
  $body = "Backup successfully ran on $($env:COMPUTERNAME).skillsnet.dk at $(Get-Date)."
  Send-MailMessage -SmtpServer "mail.nordicbackup.net" -To "support@nordicbackup.net" -From "backup@skillsnet.dk" -Body $body -Subject $subject
} catch {
  Write-Host "ERROR: $($_.error.message)" -ForegroundColor Red
  $subject = "Backup failure"
  $body = "Backup failure on $($env:COMPUTERNAME).skillsnet.dk at $(Get-Date).`r`nERROR: $($_.error.message)"
  Send-MailMessage -SmtpServer "mail.nordicbackup.net" -To "support@nordicbackup.net" -From "backup@skillsnet.dk" -Body $body -Subject $subject
}
```

## AppLocker

**GPOs:**

  - `Computer > Policies > Windows > Security > System Services`
    - **Application Identity:** Automatic start
  - `Computer > Policies > Windows > Security > Application Control Policies`
    - Create default rules
    - **Executable:** Deny `%PROGRAMFILES%\Windows NT\Accessories\wordpad.exe`

## SAM

```powershell
if ($sam.Length -gt 20) {
  Write-Host "$i. Skipping user $sam... Username invalid"
} else {
	# same logic
}
```
