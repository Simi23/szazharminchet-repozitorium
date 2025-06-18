---
title: OpenSSH Server on Windows
description: 
published: true
date: 2025-06-18T11:27:29.408Z
tags: windows
editor: markdown
dateCreated: 2025-06-18T11:27:29.408Z
---

# Installation

In Powershell, enter the following commands:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server
Add-WindowsCapability -Online -Name OpenSSH.Client
```

> Make sure the computer has internet access.
{.is-info}

# Configuration

After SSH is installed, make sure your firewall accepts incoming SSH connections.

```powershell
New-NetFirewallRule -Name 'OpenSSH-Server-In-TCP' `
  -DisplayName 'OpenSSH Server (sshd)' `
  -Enabled True `
  -Direction Inbound `
  -Protocol TCP `
  -Action Allow `
  -LocalPort 22
```

After that, start the service and set it to automatically start on boot.

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

# Key-based authentication

On the client host, create a keypair *(if you don't have one yet)*.

```bash
ssh-keygen
```

You will get two files, e.g. `id_rsa` and `id_rsa.pub`. The former is the private key, while the latter is the public key. You need to give the public key to the server to log in using public key authentication.

If this is for a regular user, copy the contents of the public key into `C:\Users\<username>\.ssh\authorized_keys`

If the key is used for an administrator account, copy the contents of the public key into `C:\ProgramData\ssh\administrators_authorized_keys`. In this case, you need to set up ACLs on the file, so only **SYSTEM** and **Administrators** have access to the file, otherwise it won't work.

After the key has been copied, you can log in using it.
