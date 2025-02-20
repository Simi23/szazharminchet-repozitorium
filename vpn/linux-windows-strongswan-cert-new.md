---
title: S2S Strongswan - RRAS (Certificate)
description: Site-to-site VPN between Windows and Linux using RRAS and Strongswan
published: true
date: 2025-02-17T12:00:21.326Z
tags: linux, windows
editor: markdown
dateCreated: 2025-02-17T11:00:47.969Z
---

# Information

The tunnel will be established between a Debian and a Windows host. Both share the network `10.0.0.0/24`, and they also have their own networks.

 - Debian shared: `10.0.0.101`
 - Debian private: `10.10.0.254/24`
 - Windows shared: `10.0.0.103`
 - Windows private: `10.30.0.254/24`

# Certificate setup

This guide assumes you have certificates ready to use. Make sure to include the **IPSec IKE Intermediate** EKU in the certificate.

> If using OpenSSL to generate certificates, this EKU is referenced with OID `1.3.6.1.5.5.8.2.2`.
{.is-info}

The guide will use two certificates, each for a machine, with the following properties:

 - DN: `C=DK, O=LEGO, CN=srv1.lego.dk`, SAN: `DNS:srv1.lego.dk`
 - DN: `C=DK, O=LEGO, CN=srv-win.test.dk`, SAN: `DNS:srv1.test.dk`
 
Certificate locations on Debian:
 - Certificate: `/etc/swanctl/x509/vpnCert.pem`
 - Key: `/etc/swanctl/private/vpnKey.pem`
 - CA Certificate: `/etc/swanctl/x509ca/win-ca.pem`

On Windows, the certificate is installed in the **Personal** (Local Machine) store.
 
> The domains are notably different, but the certificates are still from the same CA. This will not impact the negotiation, however, the machines will be referenced by their DNS names, so each host has to be able to resolve the other host by their FQDN.
{.is-info}

# Strongswan setup

This guide will use the new, `swanctl`-based configuration.

## Install Strongswan

If you install `strongswan`, you get the legacy version, so instead, other packages are needed.

```bash
apt install charon-systemd strongswan-swanctl
```

## Configure Strongswan

Edit `/etc/swanctl/swanctl.conf`.

```c
connections {
	s2s {
		local_addrs  = 10.0.0.101
		remote_addrs = srv-win.test.dk
		proposals = aes256-sha384-prfsha384-modp1024
    pools = pool1

		local {
			auth = pubkey
			certs = vpnCert.pem
		}
		remote {
			auth = pubkey
			id = "C=DK, O=LEGO, CN=srv-win.test.dk"
		}
		children {
			net {
				local_ts = 10.10.0.0/24
				remote_ts = 10.30.0.0/24
				start_action = trap
			}
		}
		version = 2
	}
}

pools {
	pool1 {
		addrs = 192.0.2.0/24
	}
}
```

Load the certs and the configuration

```bash
swanctl --load-all
```

# RRAS Setup

## Install RRAS

Install the **Remote Access** role with features ***DirectAccess and VPN (RAS)*** and ***Routing***.

Deploy **VPN only**.

When configuring RRAS, select **Custom configuration**, with **Demand-dial connections** and **LAN routing** selected.

## Tunnel configuration

Create a new demand-dial interface:

 - For the name, add the name of the other host, e.g. `srv1.lego.dk`
 - Select **VPN**
 - Select **IKEv2**
 - Enter the name of the other host, in this case, `srv1.lego.dk`.
 - Only **Route IP packets** is checked
 - Add a route for the other private network, in this case, `10.10.0.0/24`.
 - Skip **Dial-out credentials**.

Open the properties of the new interface:

 - **Options** > **Persistent connection**
 - In **Security**, select **Use machine certificates**

Now you have to configure which certificate RRAS has to use. To do this, open a PowerShell window and find the certificate first.

```ps
Get-ChildItem -Path cert:LocalMachine\My
```

Note the **Subject** column and using it, store your certificate in a variable.

```ps
$mycert = ( Get-ChildItem -Path cert:LocalMachine\My | Where-Object { $_.Subject -Like "CN=srv-win.test.dk, O=LEGO, C=DK" } )
```

> You can also use the thumbprint for this, be sure to replace **Subject** with **Thumbprint** in the `Where-Object` query.
{.is-info}

Now enter the following commands.

```ps
Set-VpnAuthProtocol -TunnelAuthProtocolsAdvertised Certificates -CertificateAdvertised $mycert
Set-VpnS2SInterface -Name "srv1.lego.dk" -Certificate $mycert
```

What this does:

 - [Set-VpnAuthProtocol](https://learn.microsoft.com/en-us/powershell/module/remoteaccess/set-vpnauthprotocol?view=windowsserver2025-ps) sets the certificate to be used when the Windows host is the responder.
 - [Set-VpnS2SInterface](https://learn.microsoft.com/en-us/powershell/module/remoteaccess/set-vpns2sinterface?view=windowsserver2025-ps) sets the certificate to be used on the specific tunnel to be used when the host is the initiator.

**Restart the RRAS service for good measure.**

# Testing

The Windows host should have a new PPP adapter with an IP address inside `192.0.2.0/24`. When pinging from Windows, it will use this address as source, which is unknown by the Debian host. Ping using the source address `10.30.0.254`.

```ps
ping -S 10.30.0.254 10.10.0.254
```

Pinging from the Debian host should work normally.