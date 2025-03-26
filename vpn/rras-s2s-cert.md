---
title: RRAS Certificate site-to-site VPN
description: RRAS Site-to-site VPN tunnel with certificate authentication
published: true
date: 2025-03-26T11:03:23.288Z
tags: windows
editor: markdown
dateCreated: 2025-03-26T11:03:23.288Z
---

# Topology

In this guide, the following topology will be used:

```
[PARIS-ROUTER] ====== {INTERNET} ====== [LYON-ROUTER]
    20.0.0.2/24                        30.0.0.2/24
```

**FQDNs:**

  - paris-router.paris.local
  - lyon-router.lyon.paris.local

# Certificate setup

Create a certificate template by copying **IPSec intermediate** template and change the following:

  - **General > Template Name:** VPN
  - **Request Handling:** Check *Include symmetric algorithms allowed by the subject*
  - **Request Handling:** Check *Allow private key to be exported*
  - **Subject Name:** Supply in the request
  - **Extensions > Application Policies:** Add *Server authentication* and *Client authentication*
  - **Extensions > Key Usage:** Check *Signature is proof of origin* and *Allow encryption of user data*

**Make sure both servers trust your root CA certificate.**

Create a certificate for each machine. Select your new template, and set the following parameters in **Subject**:

  - **Subject name > Full DN:** `CN=<SERVER_FQDN>`
  - **Subject Alternative Names:**
    - **DNS:** `<SERVER_FQDN>`
    - **DNS:** `<SERVER_IP>` (Yes, really)
    - **IPv4:** `<SERVER_IP>`

# RRAS configuration

This guide assumes you already have **RRAS installed and enabled**.

Create a **new demand-dial network interface** with the following settings:

  - **Interface name:** Other endpoint's FQDN
  - **Select** *Connect using VPN*
  - **Select** *IKEv2*
  - **Host:** Other endpoint's IP address
  - **Check** *Route IP packets*
  - Add static routes to reach the other site

Now open **properties** of the interface and in **Options > Connection Type** select *Persistent connection*. Make sure **Security > Authentication** is set to *Use machine certificates* and **verification is checked**.

The tunnel will need addressing. You have to select a *'server'* and a *'client'* endpoint. In this case, **PARIS-ROUTER** will be the **server** with a **static IP** and an IP pool to give out addresses to other peers.

On the **server** (PARIS-ROUTER), set a **static IP on the interface** in networking. For example, set `200.0.0.1`.

On the **client** (LYON-ROUTER), select *Obtain an IP address automatically* in networking.

Now, **create the pool** on the server: In **RRAS** right-click on `<SERVER_NAME> (local)` and select **Properties**. In **IPv4 > IPv4 Address assignment** select *Static pool* and create pool, e.g. `200.0.0.2 - 200.0.0.100`.

Now on **both machines**, in the same **Properties** window, go to **Security > Authentication methods** and check *Allow machine certificate authentication for IKEv2*.

Now you need to set the certificate to be used with the tunnel. For this, you need PowerShell.

First, store the certificate in a variable. **The certificates should already be imported into the Local Machine Personal store.**

```powershell
# Read output and search for your certificate
Get-ChildItem Cert:\LocalMachine\My

# Store the appropriate certificate into a variable
# In this example, it is the first certificate
$cert = (Get-ChildItem Cert:\LocalMachine\My)[0]

# You can verify by looking at the variable
$cert
```

Now, set the certificate to be used with RRAS.

```powershell
# This sets the responder config for the machine
Set-VpnAuthProtocol -TunnelAuthProtocolsAdvertised Certificates -CertificateAdvertised $cert

# This sets the initiator config for the tunnel
Set-VpnS2SInterface -Name "<TUNNEL_NAME>" -Certificate $cert
```

Now, **restart RRAS** on both machines, and the tunnel should connect. To connect the tunnel manually, be sure to connect from the *'client'* peer.