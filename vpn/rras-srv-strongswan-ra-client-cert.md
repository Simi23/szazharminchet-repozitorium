---
title: RRAS Server - Strongswan client (Certificate)
description: Remote access VPN server with RRAS Server and Strongswan Client with Certificate authentication
published: true
date: 2025-02-18T13:23:55.734Z
tags: linux, windows
editor: markdown
dateCreated: 2025-02-18T08:58:10.104Z
---

# Information

The tunnel will be established between a Debian and a Windows host. Both share the network `200.0.0.0/24`, and the Windows Server also has their own network.

 - Debian shared: `200.0.0.2`
 - Windows shared: `200.0.0.1`
 - Windows private: `10.0.0.103/24`

# Windows Setup

## Directory

In **Active Directory Users and Computers**, create a user **Security Group** called *VPN Users*. This group will hold the users you want to have remote access. Add a user to the group.

## Certificate setup

Two certificate templates will be needed for this guide. One for the **server**, and one for **user authentication**.

Open **Certification Authority**, click on **Certificate Templates** > **Manage**. Copy a server certificate (e.g. Web Server), name it *VPN Server*, and make sure it has **Server Authentication** and **IPSec IKE intermediate** EKUs (Application policies).

Copy the **User** template:

 - Name it *VPN User*
 - Raise CA and recipient compatibility level to **Windows Server 2016/Windows 10**
 - Uncheck **Publish certificate in Active Directory**
 - In **Security**, remove **Domain Users**
 - Add the **VPN Users** group, set permission to **Read, Enroll, Autoenroll**
 - In **Subject Name**, uncheck **Include e-mail name in subject name** and **E-mail name**

Create a certificate for the VPN server. For the subject name, the guide will use `C=DK, O=LEGO, CN=srv-win.test.dk`. For alternate names

 - `DNS:srv-win.test.dk`, 
 - `DNS:200.0.0.1` and 
 - `IP4:200.0.0.1`

will be used. After creating the request, submit it, then install the certificate in the **Local Machine Personal** store.

Open the **Local User** certificate manager, and create a custom request using the **VPN User** template. After submitting the request, install it in the **Local User Personal** store, then export it. Include private key, and encrypt it with a password.

Copy the resulting **pfx** file to the client using `scp`.

> If the client doesn't already have it, also copy the CA certificate.
{.is-info}

## Network Policy Server

Install the **Network Policy and Access Services** role to the server. After installation, open **Network Policy Server** console. Right click on **NPS (Local)**, and select *Register server in Active Directory*.

In the **Getting Started** section, select **Configure VPN or Dial-Up**. In the wizard:

 - Select **Virtal Private Network (VPN) Connections**, then click **Next**.
 - For clients, add localhost with IP address `127.0.0.1` (or any remote client).
 - In **Authentication Methods**, uncheck *MS-CHAPv2* and select *EAP*. Set type to *Microsoft: Smart Card or other certificate*. Click on **Configure** and select the previously created **VPN Server certificate**.
 - Select the **VPN Users** group.
 - For **IP Filters, Encryption Settings** and **Realm Name**, leave everything as **default**, then finish the wizard.

## Remote Access Server

Install the **Remote Access** role with **DirectAccess and VPN (RAS)** feature. In the configuration window, select **Deploy VPN only**.

Open the **Routing and Remote Access** management console, and select **Configure Routing and Remote Access**. Select **Custom configuration**, then check at least **VPN access**.

After setup, right click on **Ports** and select **Properties**. In **SSTP, L2TP, PPTP** and **PPPoE**, uncheck everything, so only IKEv2 and GRE are left.

By right-clicking on **Servername (local)**, open **Properties**:

 - In **Security** tab, open **Authentication methods** and make sure only **EAP** and **IKEv2** are selected.
 - In **IPv4**, select **Static address** pool, and create a pool, for example `192.0.2.1-254`.

Your server is now ready to accept connections.

# Debian Setup

Unpack the **pfx** file received from the Windows server.

```bash
openssl pkcs12 -in client.pfx -nodes -nocerts -out clientKey.pem
openssl pkcs12 -in client.pfx -nokeys -clcerts -out clientCert.pem
```

Copy the certificates to the right directories.

```bash
cp win_ca.crt /usr/local/share/ca-certificates
cp win_ca.crt /etc/swanctl/x509ca/win_ca.pem
cp clientCert.pem /etc/swanctl/x509/
cp clientKey.pem /etc/swanctl/private/
```

Edit the strongswan configuration.

```c
connections {
	home {
  	local_addrs = 200.0.0.2
    remote_addrs = 200.0.0.1
    proposals = aes256-sha384-prfsha384-modp1024
    vips = 0.0.0.0
    version = 2
    
    local {
    	auth = eap-tls
      certs = clientCert.pem
      id = "Administrator@test.dk"
    }
    remote {
    	auth = pubkey
      id = "C=DK, O=LEGO, CN=srv-win.test.dk"
    }
    children {
    	home {
      	remote_ts = 10.0.0.0/24
        start_action = start
      }
    }
  }
}
```

Load the configuration and watch the output.

```bash
swanctl --load-all
journalctl -fxeu strongswan
```

The Security Association should be established and the client should access the `10.0.0.0/24` network.

