---
title: ASP.NET SAML Application Deployment
description: Web application deployment using IIS
published: true
date: 2025-06-18T13:31:54.968Z
tags: windows
editor: markdown
dateCreated: 2025-06-18T13:31:54.967Z
---

# IIS Installation

Make sure your IIS installation has the following sub-features installed:

- .NET Extensibility 4.8
- AS<span>P.</span>NET 4.8
- ISAPI Extensions
- ISAPI Filters

# Configuration

**Create a new site.** Put all of the application files inside the root directory. Restart the server.

Inside **ADFS**, create a **New Relying Party Trust** with the following settings:

- Select **Claims aware**
- Select **Enter the data about the relying party manually**
- Enter a display name
- Do not configure token encryption certificates
- Enable support for both **WS-Fedaration** and **SAML protocols**
  - Enter the URL of the original web server you want to protect, e.g. `https://app.company.com`
- For the **Identifier**, enter the URL of the web server, e.g. `https://app.company.com` 
- For access control policy, leave **Permit everyone**
- Finish the wizard

If you want to configure extra claims, right-click on your application in the *Relying Party Trusts* folder and select **Edit Claim Issuance Policy**. Add a new rule, select **Send LDAP attributes as Claims**, then add all the **LDAP attribute - Claim name** pairs you need.