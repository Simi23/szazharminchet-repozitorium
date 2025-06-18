---
title: ADDS User import from CSV
description: ADDS User import from CSV
published: true
date: 2025-06-18T13:59:49.989Z
tags: windows
editor: markdown
dateCreated: 2025-06-18T13:41:31.451Z
---

# ADDS User import from CSV

```powershell
New-ADOragnizationalUnit -ProtectedFromAccidentalDeletion $false -Name Skills
New-ADOragnizationalUnit -ProtectedFromAccidentalDeletion $false -Name Users -Path "OU=Skills,DC=skillsnet,DC=dk"
New-ADOragnizationalUnit -ProtectedFromAccidentalDeletion $false -Name Groups -Path "OU=Skills,DC=skillsnet,DC=dk"
New-ADOragnizationalUnit -ProtectedFromAccidentalDeletion $false -Name Computers -Path "OU=Skills,DC=skillsnet,DC=dk"
New-ADOragnizationalUnit -ProtectedFromAccidentalDeletion $false -Name Servers -Path "OU=Skills,DC=skillsnet,DC=dk"
New-ADOragnizationalUnit -ProtectedFromAccidentalDeletion $false -Name Employees -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"
New-ADOragnizationalUnit -ProtectedFromAccidentalDeletion $false -Name Tech -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"
New-ADOragnizationalUnit -ProtectedFromAccidentalDeletion $false -Name Sales -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"
New-ADOragnizationalUnit -ProtectedFromAccidentalDeletion $false -Name Finance -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"
New-ADOragnizationalUnit -ProtectedFromAccidentalDeletion $false -Name Development -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"
New-ADOragnizationalUnit -ProtectedFromAccidentalDeletion $false -Name Constractors -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"

#ATTRIBUTES Firstname,LastName,samAccountName,userPrincipalName,Email,JobTitle,City,Company,Department
$path = "c:\users.csv"
$password = ConvertTo-SecureString -AsPlainText -Force "Passw0rd!"
$csv = Import-Csv -Path $path
$i = 1
foreach ($user in $csv) {
	$finame = $user.FirstName
  $lname = $user.LastName
  $funame = $finame + " " + $lname
  $sam = $finame + "." + $lname
  $pname = $user.UserPrincipalName
  $email = $user.Email
  $job = $user.JobTitle
  $city = $user.City
  $company = $user.Company
  $dep = $user.Department
  
  $exUser = Get-ADUser -Filter (SamAccountName -eq $sam -ErrorAction SilentlyContinue)
  $exGrp = Get-ADGroup -Filter (Name -eq $dep) -SearchBase "OU=Groups,OU=Skills,DC=skillsnet,DC=dk" -ErrorAction SilentlyContinue)
  
  if (!$exGrp) {
  	New-ADGroup -Name $dep -Path "OU=Groups,OU=Skills,DC=skillsnet,DC=dk" -GroupScope Global
  }
  
  if (exUser) {
  	Write-Host "$i. The $sam user exists..."
  } else {
  	New-AdUser `
    	-Name $funame `
      -GivenName $finame `
      -Surname $lname `
      -SamAccountName $sam `
      -UserPrincipalName $pname `
      -Path "OU=$dep,OU=Users,DC=skillsnet,DC=dk" `
      -Title $job `
      -AccountPassword $password `
      -Enabled $true `
      -Department $dep `
      -City $city `
      -Company $company ` 
      -EmailAddress $email `
      -DisplayName $funame
      
    Add-AdGroupMembership -Identity $dep -Members $sam
      
    Write-Host "$i. The $sam has been successfully created."
    $i++
  }
}
```
