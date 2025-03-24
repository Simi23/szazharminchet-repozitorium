---
title: RemoteApp Configuration
description: 
published: true
date: 2025-03-24T10:52:43.250Z
tags: windows
editor: markdown
dateCreated: 2025-03-24T10:52:43.250Z
---

# Role Installation

In **Add Roles**, select **Remote Desktop Services Installation** as the *Installation Type*. For the *Deployment Type*, select **Quick Start**.

Select **Session-based** *Deployment Scenario*.

Leave everything else as default.

After the role is installed, navigate to **QuickSessionCollection** in the **Remote Desktop Services** tab of **Server Manager**.

Here, you can publish additional programs and also add additional groups. *(A bug workaround is needed for principals outside of the current domain, see later.)*

After the service is started, you can access the virtual connections in the browser, at:

```
https://<IP-or-FQDN>/rdweb
```

To use a custom certificate, make sure it is installed in the **Local Machine Personal** store, then in **IIS Manager**, **edit bindings** for the virtual host, and select the certificate.

# Adding principals outside the current domain

Adding security principals to access a Session Collection will fail when those principals are outside of the current domain, even if there is trust between them.

However, there is a simple workaround: **create a domain-local security group** in the **current domain**, and add that group to the **Session Collection** **User Groups**. After that, you can **add any group as a member of this domain-local group**, and that group will have access to the session collection.
