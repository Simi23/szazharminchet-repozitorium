---
title: ES25 - ModB - 1st Solution
description: 
published: true
date: 2025-06-26T09:42:36.960Z
tags: windows, es25-windows, es25
editor: markdown
dateCreated: 2025-06-26T09:03:28.237Z
---

# ES25 - ModB - 1st Solution

[//]: <> (General)
<details>
<summary>General</summary>

- Hostname (`Rename-Computer -Name HOSTNAME`)
- IPv4 settings (`netsh int ipv4 set add Ethernet0 static add mask gateway`)
- IPv6 settings (`netsh int ipv6 set add Ethernet0 add/mask`)
  
</details>

[//]: <> (AD CS)
<details>
<summary>AD CS</summary>
  
</details>

[//]: <> (AD DS)
<details>
<summary>AD DS</summary>
  
  **DC settings**
  - `Install-WindowsFeature -Name Ad-Domain-Services, DNS -IncludeManagementTools`
  - `$password = ConvertTo-SecureString -AsPlainText -Force "Passw0rd!"`
  - `Install-ADDSForest -DomainName skillsnet.dk -SafeModePassword $password`
  
  **RODC settings**
  - DNS settings
  - `Add-Computer -DomainName skillsnet.dk`
  - `Restart-Computer`
  - `$password = ConvertTo-SecureString -AsPlainText -Force "Passw0rd!"`
  - `Install-WindowsFeature -Name Ad-Domain-Services, DNS -IncludeManagementTools`
  - `Install-ADDSDomainController -DomainName skillsnet.dk -SiteName Default-First-Site -SafeModePassword $password`
  
  **CLIENT settings**
  - DNS settings
  - `Add-Computer -DomainName skillsnet.dk`
  - `Restart-Computer`
  
</details>

[//]: <> (AD FS)
<details>
<summary>AD FS</summary>
  
</details>

[//]: <> (AD Sites)
<details>
<summary>AD Sites</summary>

>   DO IT LAST AND DON'T FORGET IT
{.is-warning}
</details>


[//]: <> (Ansible)
<details>
<summary>Ansible</summary>
  
> CREATE AND USE THE JSON
{.is-warning}
</details>

[//]: <> (Backup)
<details>
<summary>Backup</summary>
  
> USE COMMENTS AND ADD COMMENTS TO YOUR OUTPUT TOO
{.is-warning}
  ```ps
  # Variables
# Emails settings
$smtpServer = "mail.nordicbackup.net"
$from = "backup@skillsnet.dk"
$to = "support@nordicbackup.net"
$subject = ""
$body = ""
$success = $true

# Backup path
$BackupRoot = "C:\Backups"
$usersCSV = "$backupRoot\Users.csv"
$gpoBackup = "$backupRoot\GPOs"
$webBackup = "$backupRoot\Web"

try {
    # Create backup folders
    Write-Output "Create backup folders"
    New-Item -Path $BackupRoot -ItemType Directory -Force | Out-Null
    New-Item -Path $gpoBackup -ItemType Directory -Force | Out-Null
    New-Item -Path $webBackup -ItemType Directory -Force | Out-Null

    # Export users to CSV
    # FirstName,LastName,samAccountName,UserPrincipalName,Email,JobTitle,City,Company,Department
    Write-Output "Export users to CSV"
    Get-ADUser -Filter * -Properties GivenName, Surname, SamAccountName, DistinguishedName, UserPrincipalName, EmailAddress, Title, City, Company, Department `
        | Select-Object GivenName, Surname, SamAccountName, DistinguishedName, UserPrincipalName, EmailAddress, Title, City, Company, Department `
        | Export-Csv -Path $usersCSV -NoTypeInformation -Encoding UTF8


    # Export Group Policy Objects
    Write-Output "Export Group Policy Objects"
    $allGPO = Get-GPO -All
    foreach ($gpo in $allGPO) {
        $gpoName = $gpo.DisplayName
        $gpoPath = Join-Path $gpoBackup $gpoName
        New-Item -Path $gpoPath -ItemType Directory -Force | Out-Null
        Backup-GPO -Name $gpoName -Path $gpoPath -ErrorAction SilentlyContinue
    }

    # Backup IIS web root folders
    Write-Output "Backup IIS web root folders"
    Import-Module WebAdministration

    # Get all IIS sites
    $sites = Get-Website

    foreach ($site in $sites) {
        $siteName = $site.Name

        $sourcePath = $site.PhysicalPath
        $sourcePath = $sourcePath.Replace('%SystemDrive%', 'C:')
        $destinationPath = "C:\Backups\Web\$siteName"

        Write-Host "Backing up site '$siteName' from $sourcePath to $destinationPath"

        # Create destination folder
        New-Item -ItemType Directory -Path $destinationPath -Force | Out-Null

        # Copy the site files
        Copy-Item -Path $sourcePath\ -Destination $destinationPath -Recurse -Force -ErrorAction Stop
    }

    # Send success email
    Write-Output "Sending success email"
    $subject = "Backup Success on $env:COMPUTERNAME"
    $body = "The backup completed successfully on $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')."
    Send-MailMessage -From $from -To $to -Subject $subject -Body $body -SmtpServer $smtpServer
} catch {
    # Sending failure email
    Write-Output "Sending failure email"
    $succes = $false
    $subject = "Backup FAILED on $env:COMPUTERNAME"
    $body = "Backup failed on $(Get-Date). Error: $($_.Exception.Message)"
    Send-MailMessage -From $from -To $to -Subject $subject -Body $body -SmtpServer $smtpServer
    Write-Output "Backup failed on $(Get-Date). Error: $($_.Exception.Message)"
}
  ```
</details>

[//]: <> (Bitlocker)
<details>
<summary>Bitlocker</summary>
 
- `Install-WindowsFeature Bitlocker -IncludeManagementTools`
- `Enable-Bitlocker -TpmProtection "D:\"`
> Bitlocker TPM encryption doesn't work in anything else than system drive
{.is-danger}

</details>

[//]: <> (DFS + FSRM)
<details>
<summary>DFS + FSRM</summary>
  
  - `Install-WindowsFeature FS-Resource-Manager, FS-DFS-Namespace, FS-DFS-Replication -IncludeManagementTools`
  - `Enable-NetFirewallRule -DisplayGroup "Remote File Server Resource Manager Management"`

> **DFS**
> Create the NAMESPACE and it will configure the Replication for you
{.is-info}

  
> **FSRM**
> Do it from Management console, it will be faster.
{.is-info}

</details>


[//]: <> (DNS)
<details>
<summary>DNS</summary>

  > **+ CNAME Records to add**
  > <span>DC.skillsnet.</span>dk: **sso**, **ocsp**
  > <span>SRV2.skillsnet.</span>dk: **app**, **cacerts**, **crl**, **intra**, **www**
  {.is-info}

</details>

[//]: <> (IIS)
<details>
<summary>IIS</summary>
  
</details>

[//]: <> (iSCSI)
<details>
<summary>iSCSI</summary>
  
  **Target**
  - Add from server manager and get done everyting with the server manager
  - After done with settings Restart **WinTarget** and set it's *startup type* to *automatic*
  
  **Initiator**
  - -Start **MSiSCSI** and set it's *startup type* to *automatic*
  - Connect from iSCSI Initiatior management console (from tools)
</details>


[//]: <> (Firewall)
<details>
<summary>Firewall</summary>
  
</details>

[//]: <> (GPO)
<details>
<summary>GPO</summary>
  
</details>

[//]: <> (HA DHCP)
<details>
<summary>HA DHCP</summary>
  
</details>

[//]: <> (PS)
<details>
<summary>PS</summary>
  
</details>

[//]: <> (S2S - PSK)
<details>
<summary>S2S (PSK)</summary>
  
</details>

[//]: <> (S2S - Cert)
<details>
<summary>S2S (Cert)</summary>
  
</details>

[//]: <> (SMB)
<details>
<summary>SMB</summary>
  
</details>

[//]: <> (WAP)
<details>
<summary>WAP</summary>
  
</details>

[//]: <> (WEF)
<details>
<summary>WEF</summary>
  
</details>




