---
title: Dual-Hub, Dual-Stack Phase 3 DMVPN with IKEv2 PSK
description: Dual-Hub, Dual-Stack Phase 3 DMVPN with IKEv2 PSK
published: true
date: 2025-05-19T13:16:40.556Z
tags: cisco
editor: markdown
dateCreated: 2025-05-19T13:11:49.817Z
---

# Dual-Hub, Dual-Stack Phase 3 DMVPN with IKEv2 PSK


## HUB1 
> Information about routers
> Link-to internet: IPv4: 4.4.4.4/24, IPV6: 2001:DB8:4:4:4::4/64
> VPN link: IPv4: 10.100.100.1/24 IPv6: 2001:DB<span>8:10</span>0<span>:10</span>0::1/64
{.is-info}

### Crypto config
```
crypto ikev2 proposal IKE-PROP 
 encryption aes-gcm-256
 prf sha512
 group 21

crypto ikev2 policy IKE-POL 
 proposal IKE-PROP
 match fvrf any
 match address local x.x.x.x

crypto ikev2 keyring IKE-KEYRING
 peer VPN
  address 0.0.0.0 0.0.0.0
  pre-shared-key Passw0rd

crypto ikev2 profile IKE-PROF
 match fvrf any
 match address local 4.4.4.4
 match identity remote any
 authentication remote pre-share
 authentication local pre-share
 keyring local IKE-KEYRING

crypto ipsec transform-set IPSEC-TRANS esp-aes 256 esp-sha512-hmac 
 mode tunnel

crypto ipsec profile IPSEC-PROF
 set transform-set IPSEC-TRANS 
 set ikev2-profile IKE-PROF
```

### Tunnel config
```
int Tunnel1
 ip address 10.100.100.1 255.255.255.0
 ip nhrp authentication Passw0rd
 ip nhrp network-id 1000
 no ip redirects
 ipv6 address 2001:db8:100:100::1/64
 ipv6 nhrp authentication Passw0rd
 ipv6 nhrp network-id 1000
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel key 1000
 tunnel vrf DATA
 tunnel protection ipsec profile IPSEC-PROF
```


## HUB2

> Information about routers
> Link-to internet: IPv4: 1.1.1.1/24, IPV6: 2001:DB8:1:1:1::1/64
> VPN link: IPv4: 10.200.200.1/24 IPv6: 2001:DB8:200:200::1/64
{.is-info}


### Crypto config
```
crypto ikev2 proposal IKE-PROP 
 encryption aes-gcm-256
 prf sha512
 group 21

crypto ikev2 policy IKE-POL 
 proposal IKE-PROP
 match fvrf any
 match address local x.x.x.x

crypto ikev2 keyring IKE-KEYRING
 peer VPN
  address 0.0.0.0 0.0.0.0
  pre-shared-key Passw0rd

crypto ikev2 profile IKE-PROF
 match fvrf any
 match address local 1.1.1.1
 match identity remote any
 authentication remote pre-share
 authentication local pre-share
 keyring local IKE-KEYRING

crypto ipsec transform-set IPSEC-TRANS esp-aes 256 esp-sha512-hmac 
 mode tunnel

crypto ipsec profile IPSEC-PROF
 set transform-set IPSEC-TRANS 
 set ikev2-profile IKE-PROF
```

### Tunnel config
```
int Tunnel2
 ip address 10.200.200.1 255.255.255.0
 no ip redirects 
 ip nhrp authentication Passw0rd
 ip nhrp network-id 2000
 ipv6 address 2001:db8:200:200::1/64
 ipv6 nhrp authentication Passw0rd
 ipv6 nhrp network-id 2000
 ipv6 nhrp map multicast dynamic
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel key 2000
 tunnel vrf DATA
 tunnel protection ipsec profile IPSEC-PROF

```

## SPOKE1

> Information about routers
> Link-to internet: IPv4: PPPoE, IPV6: 2001:DB8:2:2:2::2/64
> VPN link(Tunnel 1): IPv4: 10.100.100.2/24 IPv6: 2001:DB<span>8:10</span>0<span>:10</span>0::2/64
> VPN link(Tunnel 2): IPv4: 10.200.200.2/24 IPv6: 2001:DB8:200:200::2/64
{.is-info}


### Crypto config
```
crypto ikev2 proposal IKE-PROP 
 encryption aes-gcm-256
 prf sha512
 group 21

crypto ikev2 policy IKE-POL 
 proposal IKE-PROP

crypto ikev2 keyring IKE-KEYRING
 peer VPN
  address 0.0.0.0 0.0.0.0
  pre-shared-key Passw0rd

crypto ikev2 profile IKE-PROF
 match fvrf any
 match identity remote any
 authentication remote pre-share
 authentication local pre-share
 keyring local IKE-KEYRING

crypto ipsec transform-set IPSEC-TRANS esp-aes 256 esp-sha512-hmac 
 mode tunnel

crypto ipsec profile IPSEC-PROF
 set transform-set IPSEC-TRANS 
 set ikev2-profile IKE-PROF
```

### Tunnel config
```
interface Tunnel1
 ip address 10.100.100.2 255.255.255.0
 no ip redirects
 ip nhrp authentication Passw0rd
 ip nhrp network-id 1000
 ip nhrp nhs 10.100.100.1 nbma 4.4.4.4 multicast
 ipv6 address 2001:db8:100:100::2/64
 ipv6 nhrp authentication Passw0rd
 ipv6 nhrp network-id 1000
 ipv6 nhrp nhs 2001:db8:100:100::1 nbma 4.4.4.4 multicast
 tunnel source Dialer1
 tunnel mode gre multipoint
 tunnel key 1000
 tunnel protection ipsec profile IPSEC-PROF shared


interface Tunnel2
 ip address 10.200.200.2 255.255.255.0
 no ip redirects
 ip nhrp authentication Passw0rd
 ip nhrp nhs 10.200.200.1 nbma 1.1.1.1 multicast
 ip nhrp network-id 2000
 ipv6 address 2001:db8:200:200::2/64
 ipv6 nhrp authentication Passw0rd
 ipv6 nhrp network-id 2000
 ipv6 nhrp nhs 2001:db8:200:200::1 nbma 1.1.1.1 multicast
 tunnel source Dialer1
 tunnel mode gre multipoint
 tunnel key 2000
 tunnel protection ipsec profile IPSEC-PROF shared
```

## SPOKE2

> Information about routers
> Link-to internet: IPv4: PPPoE, IPV6: 2001:DB8:3:3:3::3/64
> VPN link(Tunnel 1): IPv4: 10.100.100.3/24 IPv6: 2001:DB<span>8:10</span>0<span>:10</span>0::3/64
> VPN link(Tunnel 2): IPv4: 10.200.200.3/24 IPv6: 2001:DB8:200:200::3/64
{.is-info}


### Crypto config
```
crypto ikev2 proposal IKE-PROP 
 encryption aes-gcm-256
 prf sha512
 group 21

crypto ikev2 policy IKE-POL 
 proposal IKE-PROP

crypto ikev2 keyring IKE-KEYRING
 peer VPN
  address 0.0.0.0 0.0.0.0
  pre-shared-key Passw0rd

crypto ikev2 profile IKE-PROF
 match fvrf any
 match identity remote any
 authentication remote pre-share
 authentication local pre-share
 keyring local IKE-KEYRING

crypto ipsec transform-set IPSEC-TRANS esp-aes 256 esp-sha512-hmac 
 mode tunnel

crypto ipsec profile IPSEC-PROF
 set transform-set IPSEC-TRANS 
 set ikev2-profile IKE-PROF
```

### Tunnel config
```
interface Tunnel1
 ip address 10.100.100.3 255.255.255.0
 no ip redirects
 ip nhrp authentication Passw0rd
 ip nhrp network-id 1000
 ip nhrp nhs 10.100.100.1 nbma 4.4.4.4 multicast
 ipv6 address 2001:db8:100:100::3/64
 ipv6 nhrp authentication Passw0rd
 ipv6 nhrp network-id 1000
 ipv6 nhrp nhs 2001:db8:100:100::1 nbma 4.4.4.4 multicast
 tunnel source Dialer1
 tunnel mode gre multipoint
 tunnel key 1000
 tunnel protection ipsec profile IPSEC-PROF shared

interface Tunnel2
 ip address 10.200.200.3 255.255.255.0
 no ip redirects
 ip nhrp authentication Passw0rd
 ip nhrp nhs 10.200.200.1 nbma 1.1.1.1 multicast
 ip nhrp network-id 2000
 ipv6 address 2001:db8:200:200::3/64
 ipv6 nhrp authentication Passw0rd
 ipv6 nhrp network-id 2000
 ipv6 nhrp nhs 2001:db8:200:200::1 nbma 1.1.1.1 multicast
 tunnel source Dialer1
 tunnel mode gre multipoint
 tunnel key 2000
 tunnel protection ipsec profile IPSEC-PROF shared
```