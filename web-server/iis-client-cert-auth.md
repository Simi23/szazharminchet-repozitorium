---
title: IIS Client certificate authentication
description: 
published: true
date: 2025-06-20T09:43:56.947Z
tags: windows
editor: markdown
dateCreated: 2025-06-20T09:43:56.947Z
---

# Prerequisites

Make sure the **IIS** role on the server has the **IIS > Web Server > Security > IIS Client Certificate Mapping Authentication** feature installed.

> There is also a **~~IIS~~ Client Certificate Mapping Authentication** feature (without the **IIS** part) available, be sure not to mix them up!
{.is-warning}

You should have a running website with an HTTPS binding, with a certificate that is trusted by your clients.

# Configuration

This guide will describe the **many-to-one binding** feature of the certificate mapping.

> Before you configure anything related to the certificate authentication, make sure your **site binding has TLS 1.3 disabled** because the TLS 1.3 client certificate authentication is not implemented correctly in Windows! Otherwise you will get a *Connection Refused* error in the browser!
{.is-danger}

In **IIS Manager**, select the site you want to protect and open **Configuration Editor**. In the *Section* dropdown, navigate to **system.webServer > security > authentication > iisClientCertificateMappingAuthentication**. Select **ApplicationHost.config** in the *From* dropdown.

Set **Enabled** to ***true***. Set **manyToOneCertificateMappingsEnabled** to ***true***. Open the **manyToOneMappings** object.

**Add a new collection:**

- Define a **name/description** for the collection.
- Populate the **userName/password** fields. The user specified here will be the user that the matching certificates will be mapped to.
- Create a rule for matching certificates.
  - The **certificate field** is a full field of the certificate, can be either **Issuer** or **Subject**.
  - The **certificate subfield** can be a common LDAP subfield, e.g. CN, OU, O, L, S, C
  - **MatchCriteria** is the string which the field/subfield will be compared against.

**Apply all changes** and go back to the IIS console. In your site settings, open **SSL Settings** and click **Require SSL**. In the **Client certificates** list, choose **Require**.

In the site settings, open **Authentication**. **Disable every item.**

**Restart the server**, than connect to the website to test it.