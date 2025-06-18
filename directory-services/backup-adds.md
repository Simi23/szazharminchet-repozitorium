---
title: Backup ADDS
description: Backup ADDS
published: true
date: 2025-06-18T13:43:44.399Z
tags: windows
editor: markdown
dateCreated: 2025-06-18T13:43:44.399Z
---

# Backup ADDS and IIS


```powershell
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

    
    # Copy items to iSCSI disk
    Copy-Item -Path $BackupRoot -Destination "b:\" -Recurse -Force
    
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