---
title: Playbook: S2S PSK (G)
description: S2S VPN using pre-shared-key with Ansible. Including EIGRP routing.
published: true
date: 2025-03-01T14:00:31.212Z
tags: ansible, cisco
editor: markdown
dateCreated: 2025-03-01T13:58:55.638Z
---

# S2S VPN + EIGRP

## Playbook

```yaml

```

## Inventory

```yaml
all:
  vars:
    ansible_user: ssh
    ansible_password: Passw0rd
    ansible_connection: ansible.netcommon.network_cli
    ansible_network_os: cisco.ios.ios

border_routers:
  hosts:
    S1-BORDER.lego.dk:
      ansible_host: 172.20.0.12
      public_ip: 1.1.1.0
      public_ipv6: 2001:db8:1:1:1::0
      remote_public_ip: 2.2.2.1
      remote_public_ipv6: 2001:DB8:2:2:2::1
      remote_public_ipv6_: 2001:DB8:2:2:2::1/64

    S2-BORDER.lego.dk:
      ansible_host: 172.20.0.14
      public_ip: 2.2.2.1
      public_ipv6: 2001:db8:2:2:2::1
      remote_public_ip: 1.1.1.0
      remote_public_ipv6: 2001:DB8:1:1:1::0
      remote_public_ipv6_: 2001:DB8:1:1:1::0/64
      
  vars:
    ospf_passive:
      - GigabitEthernet0/0
      - GigabitEthernet0/2
      - Loopback0
      - Tunnel0
    
    eigrp_v6_network:
      - name: Tunnel0
        as : 10

    eigrp_as: 10
    eigrp_passive:
      - GigabitEthernet0/0
      - GigabitEthernet0/1
      - GigabitEthernet0/2
    eigrp_network:
      - network: 10.0.0.0
        wildcard_mask: 0.0.0.255

    psk: Passw0rd
    tunnel_source: GigabitEthernet0/0
    IKEv2_keyring: IKEv2-KEYRING
    IKEv2_keyring_peer: VPN-PEER
    IKEv2_proposal: IKEv2-PROPOSAL
    IKEv2_policy: IKEv2-POLICY
    IKEv2_profile: IKEv2-PROFILE
    IPSEC_transform_set: IPSEC-TRANS
    IPSEC_profile: IPSEC-PROFILE
```