---
title: Strongswan Server - Windows Native Client (Certificate - Legacy)
description: Strongswan IKEv2 Remote Access Server with Certificate Authentication, and Windows 11 Native Client
published: true
date: 2025-03-02T09:52:40.077Z
tags: 
editor: markdown
dateCreated: 2025-02-18T13:51:11.059Z
---

# Information

The tunnel will be established between a Debian and a Windows host. Both share the network `200.0.0.0/24`, and the Linux Server also has their own network.

 - Debian shared: `200.0.0.2`
 - Windows shared: `200.0.0.1`
 - Debian private: `10.0.0.102/24`

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

Create a certificate, key pair for the VPN server on the Linux from the Root CA. Put the key into `/etc/ipsec.d/private/` and the cert into `/etc/ipsec.d/certs/` folder.
 

After creating the request, submit it, then install the certificate in the **Local Machine Personal** store.

Open the **Local User** certificate manager, and create a custom request using the **VPN User** template. After submitting the request, install it in the **Local User Personal** store, then export it. Include private key, and encrypt it with a password.

Copy the resulting **pfx** file to the windows client using `scp`.

> If the client doesn't already have it, also copy the CA certificate.
{.is-info}

## Client setup

Create a new VPN connection

- Name it **Strongswan-VPN**
- Give it your servers FQDN (what's in your cert)
- Set VPN type to **IKEv2**
- Set Type of sign-in info to **Certificate**

Save your configuration and go to advanded options. On this tab click on edit at More VPN properties

- On security tab check **Use machine certificates**

Apply it, and your windows side configurations are done.

# Debian Setup

```bash
apt install strongswan strongswan-starter

Edit the `ipsec.conf` configuration.

```c
conn LINSRV
        keyexchange=ikev2

        left=172.20.0.2
        leftsubnet=0.0.0.0/0
        leftcert=server.crt
        leftid="CN=LIN-SRV-11.lego.dk"

        right=%any
        rightsourceip= 10.255.255.0/24

        authby=rsasig

        ike=aes256-sha2_256-modp1024
        esp=aes256-sha2_256
        auto=start
```

Edit the `ipsec.secrets` configuration.

```c
: RSA server.key
```

Load the configuration and watch the output.

```bash
systemctl restart strongswan-starter
```

The Security Association should be established and the client should access the `10.0.0.0/24` network.

Now you can click on connect on Windows side, and check the connection on Linux side.

```bash
ipsec status
```

