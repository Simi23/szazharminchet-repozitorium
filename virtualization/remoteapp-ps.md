---
title: RemoteApp with PowerShell
description: RemoteApp (RDS) configuration with PowerShell scritps
published: true
date: 2025-05-29T09:25:22.280Z
tags: windows, powershell
editor: markdown
dateCreated: 2025-05-29T09:25:22.280Z
---

# RemoteApp PowerShell

> Install these features on the servers where you want (You can distribute them into other servers too).
{.is-info}

```
Install-WindowsFeature `
	RDS-RD-Server `
  RDS-Licensing `
  RDS-Connection-Broker `
  RDS-Web-Access `
  -IncludeManagementTools
```

> Restart the computer, to make RDS configurable.
{.is-info}

```
Restart-Computer
```

> Deploy the RDS session, you can set these roles to different computers, and you can run this powershell commandlet from anywhere the domain, but just from native PowerShell.
{.is-info}

> DON'T DO THESE STEPS FROM POWERSHELL REMOTE BECAUSE IT WILL NOT WORK!
{.is-danger}

```
New-RDSessionDeployment `
	-ConnectionBroker DEV-SRV.lego.dk `
  -SessionHost DEV-SRV.lego.dk `
  -WebAccessServer DEV-SRV.lego.dk
```

> Enable Licensing to per-user.
{.is-info}

```
Set-RDLicenseConfiguration -LicenseServer DEV-SRV.lego.dk -Mode PerUser -Force
```
```
Restart-Computer
```

> If you have certificates put the pfx-s to a spot where the RDS server can reach them. And run these commands for this 3 service:
> RDWebAccess
> RDPublishing
> RDRedirector
{.is-info}

```
# Pfx file password
$Password = ConvertTo-SecureString -AsPlainText "Passw0rd" -Force

Set-RDCertificate -Role RDWebAccess -ImportPath C:\ca\pfx.pfx -Password $Password -ConnectionBroker DEV-SRV.lego.dk
```

> Create a collection on the SessionHost for later configuration
{.is-info}

```
New-RDSessionCollectinon -CollectionName "User_RemoteApp_Collection" -SessionHost DEV-SRV.lego.dk -ConnectionBroker DEV-SRV.lego.dk
```


> Add the User range who can access the collection
{.is-info}

```
Set-RDSessionCollectionConfiguration -CollectionName "User_RemoteApp_Collection" -UserGroup "LEGO\Domain Users"
```

> Publish Remote Apps
{.is-info}

```
New-RDRemoteApp -CollectionName "User_RemoteApp_Collection" -DisplayName "Notepad if we installed minimal windows 10" -FilePath "C:\Windows\System32\notepad.exe"
