---
title: Strongswan Server - Windows Native Client (Certificate)
description: Strongswan IKEv2 Remote Access Server with Certificate Authentication, and Windows 11 Native Client
published: true
date: 2025-02-18T13:51:00.369Z
tags: linux, windows
editor: markdown
dateCreated: 2025-02-18T13:49:29.313Z
---

# Information

There will be two machines used in this guide: a **Debian server** running Strongswan and a **Windows 11 client** machine. They share a public network and the Debian server also has a private network.

 - Windows Client: `200.0.0.5/24`
 - Debian Server: `200.0.0.10/24`
 - Debian private network: `10.0.0.102/24`

# Certificates

Each machine will need a certificate with the following parameters:

 - **Debian Server**
   - The **DN/SAN** of the certificate has to **contain the hostname** that will be used to connect from the Windows Client, e.g. `srv2.lego.dk` -> `C=DK, O=LEGO, CN=srv2.lego.dk`
   - The certificate must include the following **EKUs/Application Policies**
     - **Server Authentication**
     - **IPSec IKE intermediate** (`1.3.6.1.5.5.8.2.2`)
 - **Windows Client**
   - The certificate must include the following EKU/Application Policy
     - **Client Authentication**

Copy the right certs and the CA cert to the machines, and make sure it is trusted by the system.

# Strongswan Setup

Install the required packages. This guide will use the new, `swanctl`-style strongswan configuration.

```bash
apt install charon-systemd strongswan-swanctl
```

Copy the CA certificate and the server certificate/key to the correct location:

```bash
cp CAcert.pem /etc/swanctl/x509ca/
cp serverCert.pem /etc/swanctl/x509/
cp serverKey.pem /etc/swanctl/private/
```

Edit the strongswan config, `/etc/swanctl/swanctl.conf`.

```c
connections {
	remoteaccess {
		pools = ra_pool
		proposals = aes256-sha384-prfsha384-modp1024
		local {
			auth = pubkey
			certs = serverCert.pem
			id = srv2.lego.dk
		}
		remote {
			auth = pubkey
		}
		children {
			ra {
				local_ts = 10.0.0.0/24
			}
		}
	}
}

pools {
	ra_pool {
		addrs = 192.0.2.0/24
	}
}

# Include config snippets
include conf.d/*.conf
```

Load the configuration, and monitor the logs for the connection coming from the client.

```bash
swanctl --load-all
journalctl -fxeu strongswan
```

# Windows Setup

This guide will use the native Windows IKEv2 client. Make sure you have the certificates imported into the right stores. The client certificate should be installed in the Local Machine Personal store.

In **Settings** > **Network & internet** > **VPN** click ***Add VPN***.

Enter a **connection name**, enter the **hostname of the VPN server** (make sure it matches the server certificate). For **VPN Type**, select ***IKEv2***. For **Type of sign-in info**, select ***Certificate***.

After creating the connection, open the dropdown menu and click **Advanced options**.

![winvpn.png](/winvpn.png)

Click **More VPN Properties** > **Edit**. In the **Security** tab, select **Machine Certificates**. Save the settings and connect.

