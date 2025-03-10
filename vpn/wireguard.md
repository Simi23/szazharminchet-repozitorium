---
title: Wireguard VPN
description: Wireguard Remote Access VPN Configuration guide
published: true
date: 2025-03-10T09:37:20.308Z
tags: linux
editor: markdown
dateCreated: 2025-03-10T09:27:56.634Z
---

# Server setup

Install the required packages.

```bash
apt install wireguard wireguard-tools
```

Generate a keypair for the server.

```bash
wg genkey > server.key
wg pubkey < server.key > server.pub
```

Generate a pre-shared key for the connection.

```bash
wg genpsk > psk
```

Create the Wireguard interface config in the following file: <kbd>/etc/wireguard/wg0.conf</kbd>

```ini
[Interface]
Address = 10.1.30.1/24
Address = 2001:db8:1001:30::1/64
ListenPort = 51820
PrivateKey = <INSERT SERVER PRIVATE KEY HERE>
```

> The `Address` field specifies the address of the tunnel interface. You can specify multiple, just like in the example.
> 
> The listening port is a randomly selected port by default.
{.is-info}

For any peer, add the following config to the <kbd>wg0.conf</kbd> file.

```ini
[Peer]
PublicKey = <INSERT PEER PUBLIC KEY>
PresharedKey = <INSERT PRE-SHARED KEY>
AllowedIPs = 10.1.30.2/32, 2001:db8:1001:30::2/128
```

You have to have the public key of the client already.

The `AllowedIPs` field specifies the routes that will be installed towards the given peer.

Save the file and bring up the interface.

```bash
wg-quick up wg0
```

Enable the service so that the interface is brought up automatically upon reboot.

```bash
systemctl enable wg-quick@wg0.service
```

# Client setup

Install the package, then create a keypair for the client just like on the server. The pre-shared key will have to be copied from the server to the client. Also, you need to copy the public keys to the peer machines.

After that, create the client config file at <kbd>/etc/wireguard/wg0.conf</kbd>

```ini
[Interface]
Address = 10.1.30.2/24
Address = 2001:db8:1001:30::2/64
PrivateKey = <INSERT CLIENT PRIVATE KEY HERE>

[Peer]
PublicKey = <INSERT SERVER PUBLIC KEY>
PresharedKey = <INSERT PRE-SHARED KEY>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = 1.1.1.10:51820
```

> Specifying `0.0.0.0/0, ::/0` for the allowed IPs will route all traffic through the tunnel.
{.is-info}

The same commands can be utilized to bring up the tunnel.

## Setup with NetworkManager

To import your Wireguard config into NetworkManager, use the following command:

```bash
nmcli connection import type wireguard file /etc/wireguard/wg0.conf
```
