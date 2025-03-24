---
title: User creation in PowerShell
description: PowerShell script for creating users in AD DS
published: true
date: 2025-03-24T10:31:44.501Z
tags: windows, powershell
editor: markdown
dateCreated: 2025-03-24T10:31:44.501Z
---

# User creation script

This script takes a `-count <n>` argument and creates **n** users in all of the groups specified in the script. The OU and the security groups have to be created in advance.

```powershell
Param(
    [int] $count
)

$password = ConvertTo-SecureString -AsPlainText -Force "Passw0rd"
$longpassword = ConvertTo-SecureString -AsPlainText -Force "Passw0rd@Passw0rd"

$basepath = ",dc=paris,dc=local"

$groups = @(
    @{username='mkt';ou='MKT';password=$password;group="cn=MKT,ou=MKT$($basepath)"},
    @{username='sales';ou='SALES';password=$password;group="cn=SALES,ou=SALES$($basepath)"},
    @{username='tech';ou='TECH';password=$password;group="cn=TECH,ou=TECH$($basepath)"},
    @{username='hr';ou='HR';password=$longpassword;group="cn=HR,ou=HR$($basepath)"}
)

foreach ($group in $groups) {
    for ($i = 1; $i -le $count; $i++) {
        $username = "$($group.username)$($i)"
        
        if (Get-ADUser -Filter {SamAccountName -eq $username}) {
            Write-Host "Skipping user $($username)"
            continue
        }

        Write-Host "Creating user $($username)"
        
        $user = New-ADUser -Name $username `
            -SamAccountName $username `
            -PassThru `
            -Path "ou=$($group.ou)$($basepath)" `
            -PasswordNeverExpires $true `
            -ChangePasswordAtLogon $false `
            -AccountPassword $group.password `
            -Enabled $true

        Add-ADGroupMember -Identity $group.group -Members $user
    }
}
```

For example, you can call this script like this:

```powershell
./create_user.ps1 -count 5
```

This way, the script will create 5 users in each group.