---
title: IIS Reverse proxy
description: Configure a reverse proxy with TLS offload and HTTPS redirection
published: true
date: 2025-07-17T08:16:34.904Z
tags: windows
editor: markdown
dateCreated: 2025-07-17T08:16:34.904Z
---

# Prerequisites

Make sure to complete these installations in the following order:

- **IIS** with ASP<span>.N</span>ET role
- **URL Rewrite** module for IIS
- **Application Request Routing** module for IIS

This guide assumes the following variables:

- The **backend server** is reachable via `http://localapp.org`
- The **proxy** will host this website as `https://publicapp.com`

# Configuration

Open **IIS Manager**, click on the host *(server name)*, and open the **Application Request Routing Cache** menu. On the right, click **Server Proxy Settings**. Check **Enable proxy**.

Configure your site. This should bind to the publicly available URL. Create a binding for both schemes, in this case:

- `http://publicapp.com`, and
- `https://publicapp.com`

Now in the site config, open the URL Rewrite menu. Create the following rules:

- **HTTPS Redirect rule**
  - **Template:** Inbound rule > Blank rule
  - **Name:** HTTPS Redirect
  - **Match URL**
    - **Requested URL:** Matches the pattern
    - **Using:** Wildcards
    - **Pattern:** `*`
    - **Check** *Ignore case*
  - **Conditions**
    - **Logical grouping:** Match all
    - **Add a condition:**
      - **Input:** `{HTTPS}`
      - **Type:** Matches the pattern
      - **Pattern:** `OFF`
      - **Check** *Ignore case*
    - **Uncheck** *Track capture groups across conditions*
   - **Action**
     - **Action type:** Redirect
     - **Redirect URL:** `https://{HTTP_HOST}{REQUEST_URI}`
     - **Uncheck** *Append query string*
     - **Redirect type:** 302 Found
- **Reverse proxy rule**
  - **Template:** Inbound and Outbound Rules > Reverse Proxy
  - **Server name:** `localapp.org`
  - **Check** *Enable SSL Offloading*
  - **Check** *Rewrite the domain names of the links in HTTP responses*
    - **From:** `localapp.org`
    - **To:** `publicapp.com`

**Restart** the IIS server. HTTPS scheme redirection and reverse proxying should now work correctly.