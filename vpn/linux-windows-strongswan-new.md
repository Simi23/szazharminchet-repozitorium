---
title: S2S Strongswan - RRAS (PSK)
description: Site-to-site VPN between Windows and Linux using RRAS and Strongswan
published: true
date: 2025-02-13T15:55:12.902Z
tags: linux, windows
editor: markdown
dateCreated: 2025-02-13T15:41:58.052Z
---

# Information

The tunnel will be established between a Debian and a Windows host. Both share the network `10.0.0.0/24`, and they also have their own networks.

 - Debian shared: `10.0.0.101`
 - Debian private: `10.10.0.254/24`
 - Windows shared: `10.0.0.103`
 - Windows private: `10.30.0.254/24`

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
		remote_addrs = 10.0.0.103
		proposals = aes256gcm16-prfsha384-modp1024

		local {
			auth = psk
			id = 10.0.0.101
		}
		remote {
			auth = psk
			id = 10.0.0.103
		}
		children {
			net {
				local_ts = 10.10.0.0/24
				remote_ts = 10.30.0.0/24
				start_action = start
			}
		}
		version = 2
	}
}

secrets {
	ike1 {
		id = 10.0.0.101
		id = 10.0.0.103
		secret = "Passw0rd"
	}
}
```

Load the configuration

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

 - Enter a name
 - Select **VPN**
 - Select **IKEv2**
 - Enter the IP address of the other host, in this case, `10.0.0.101`.
 - Only **Route IP packets** is checked
 - Add a route for the other private network, in this case, `10.10.0.0/24`.
 - Skip **Dial-out credentials**.

Open the properties of the new interface:

 - **Options** > **Persistent connection**
 - In **Security**, select **Use preshared key for authentication** and enter the key, in this case, `Passw0rd`.

Open RRAS Properties window:

 - In **Security**, check **Allow custom IPSec policy for L2TP/IKEv2** connection and enter the pre-shared key, in this case, `Passw0rd`.

Restart the RRAS service for good measure.