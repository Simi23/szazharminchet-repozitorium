---
title: IOS Site-to-site VPN IKEv2 PSK
description: Cisco IOS Site-to-site route-based VPN with IKEv2 PSK authentication
published: true
date: 2025-02-25T10:17:17.948Z
tags: cisco
editor: markdown
dateCreated: 2025-02-25T10:17:17.948Z
---

# Configuration

Create the IKEv2 keyring.

```
crypto ikev2 keyring KEYRING
  peer PEER_NAME
    address PEER_ADDRESS
    pre-shared-key Passw0rd
```

Create an IKEv2 proposal.

```
crypto ikev2 proposal PROPOSAL
  encryption aes-gcm-256
  prf sha512
  group 21
```

Create an IKEv2 policy which links the proposal.

```
crypto ikev2 policy POLICY
  proposal PROPOSAL
```

Create the IKEv2 profile.

```
crypto ikev2 profile PROFILE
  authentication local pre-share
  authentication remote pre-share
  keyring local KEYRING
  match address local LOCAL_ADDRESS
  match remote identity address REMOTE_ADDRESS
```

Create an IPsec transform set.

```
crypto ipsec transform-set TSET esp-aes 256 esp-sha512-hmac
  mode tunnel
```

Create an IPsec profile, link the transform set and the IKEv2 profile.

```
crypto ipsec profile PROFILE
  set transform-set TSET
  set ikev2-profile PROFILE
```

Finally, apply the IPsec profile to the tunnel interface.

```
interface Tunnel1
  tunnel protection ipsec profile PROFILE
```

# Full configuration

```
crypto ikev2 keyring KEYRING
  peer PEER_NAME
    address PEER_ADDRESS
    pre-shared-key Passw0rd

crypto ikev2 proposal PROPOSAL
  encryption aes-gcm-256
  prf sha512
  group 21

crypto ikev2 policy POLICY
  proposal PROPOSAL

crypto ikev2 profile PROFILE
  authentication local pre-share
  authentication remote pre-share
  keyring local KEYRING
  match address local LOCAL_ADDRESS
  match remote identity address REMOTE_ADDRESS

crypto ipsec transform-set TSET esp-aes 256 esp-sha512-hmac
  mode tunnel

crypto ipsec profile PROFILE
  set transform-set TSET
  set ikev2-profile PROFILE

interface Tunnel1
  tunnel protection ipsec profile PROFILE
```