---
title: VPN
description: VPN Guides
published: true
date: 2025-05-19T13:02:02.072Z
tags: 
editor: markdown
dateCreated: 2025-03-03T10:43:56.838Z
---

# Cisco guides

- [Site-to-site VPN with IKEv2 PSK](/vpn/cisco-ikev2-psk)
- [Dual-Hub, Dual-Stack Phase 3 DMVPN with IKEv2 PSK](//cisco-dmvpn-dual-psk)

# Server guides

## Site-to-site VPN

 - **Strongswan**-**RRAS** with **pre-shared** authentication
   - [`swanctl` config](/vpn/linux-windows-strongswan-new)
   - [`ipsec.conf` config](/vpn/s2s-strongswan-rras-old-psk)
 - **Strongswan**-**RRAS** with **certificate** authentication
   - [`swanctl` config](/vpn/linux-windows-strongswan-cert-new)
   - [`ipsec.conf` config](/vpn/s2s-strongswan-rras-old-cert)
 - **[RRAS-RRAS](/vpn/rras-s2s-cert)** with **certificate** authentication

## Remote Access VPN

 - **RRAS server** and **Strongswan client** with **certificate authentication**
   - [`swanctl` config](/vpn/rras-srv-strongswan-ra-client-cert)
   - [`ipsec.conf` config](/vpn/rras-strong-cl-cert-legacy)
 - **Strongswan server** and **Windows built-in IKEv2 client** with **certificate authentication**
   - [`swanctl` config](/vpn/strongswan-srv-windows-client-cert)
   - [`ipsec.conf` config](/vpn/win-clt-strong-srv-cert-legacy)
 - [**Wireguard**](/vpn/wireguard) with **public key** and **pre-shared key** authentication