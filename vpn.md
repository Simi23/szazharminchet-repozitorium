---
title: VPN
description: VPN Guides
published: true
date: 2025-03-03T11:04:16.152Z
tags: 
editor: markdown
dateCreated: 2025-03-03T10:43:56.838Z
---

# Server guides

## Site-to-site VPN

 - **Strongswan**-**RRAS** with **pre-shared** authentication
   - [`swanctl` config](/vpn/linux-windows-strongswan-new)
   - [`ipsec.conf` config](/vpn/s2s-strongswan-rras-old-psk)
 - **Strongswan**-**RRAS** with **certificate** authentication
   - [`swanctl` config](/vpn/linux-windows-strongswan-cert-new)
   - [`ipsec.conf` config](/vpn/s2s-strongswan-rras-old-cert)

## Remote Access VPN

 - **RRAS server** and **Strongswan client** with **certificate authentication**
   - [`swanctl` config](/vpn/rras-srv-strongswan-ra-client-cert)
   - [`ipsec.conf` config](/vpn/rras-strong-cl-cert-legacy)
 - **Strongswan server** and **Windows built-in IKEv2 client** with **certificate authentication**
   - [`swanctl` config](/vpn/strongswan-srv-windows-client-cert)
   - [`ipsec.conf` config](/vpn/win-clt-strong-srv-cert-legacy)

# Cisco guides

- [Site-to-site VPN with IKEv2 PSK](/vpn/cisco-ikev2-psk)