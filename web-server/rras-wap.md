---
title: Web Application Proxy
description: 
published: true
date: 2025-06-18T14:23:16.050Z
tags: windows
editor: markdown
dateCreated: 2025-06-18T14:23:16.050Z
---

# Prerequisites

**AD Federation Services is directly required** by Web Application Proxy, so if you don't have it installed yet, [**do it now**](/directory-services/adfs-setup).

The certificate you have created for ADFS can be used here, so **export it** (with the key included) and copy it to the WAP server. Make sure to install the certificate, too.

# Installation

Before installing WAP, **TLS 1.3 needs to be disabled** because of a bug in the certificate authentication implementation. Open **Registry Editor** and create the following keys/attributes:

```
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.3\Client]

"DisabledByDefault"=dword:00000001
"Enabled"=dword:00000000
```

**Restart the computer.**

Now you can install WAP. In **Server Manager**, add the **Routing and Remote Access** role with the **Web Application Proxy** feature selected.

Open the configuration wizard.

- Enter the federation service name, e.g. `sso.company.com`.
- Enter the user name of a Local Administrator on the AD FS server. If the AD FS server is a DC too, this can be just a Domain Administrator. You can use SAM Account Name, e.g. `COMPANY\USER`.
- Select the certificate you have installed.

**Finish the wizard.** Web Application Proxy should now be working.