---
title: Playbook: S2S PSK (G)
description: S2S VPN using pre-shared-key with Ansible. Including EIGRP routing.
published: true
date: 2025-03-01T14:02:47.116Z
tags: ansible, cisco
editor: markdown
dateCreated: 2025-03-01T13:58:55.638Z
---

# S2S VPN + EIGRP

## Playbook

```yaml
---
- name: Configure S2S GRE Tunnel (PSK)
  hosts: border_routers
  gather_facts: false
  tasks:
    # | Configure IKEv2, IPSEC |
    - name: Configure IKEv2 keyring
      cisco.ios.ios_config:
        lines:
          - address {{ remote_public_ip }} 255.255.255.0
          - pre-shared-key {{ psk }}
        parents:
          - crypto ikev2 keyring {{ IKEv2_keyring }}
          - peer {{ IKEv2_keyring_peer }}
      register: keyring
      notify: Save
    
    - name: Configure IKEv2 proposal
      cisco.ios.ios_config:
        lines:
          - encryption aes-cbc-256
          - integrity sha512
          - group 16
        parents: 
          - crypto ikev2 proposal {{ IKEv2_proposal }}
      notify: Save
    
    - name: Configure IKEv2 policy
      cisco.ios.ios_config:
        lines:
          - proposal {{ IKEv2_proposal }}
        parents:
          - crypto ikev2 policy {{ IKEv2_policy }}
      notify: Save

    - name: Configure IKEv2 profile
      cisco.ios.ios_config:
        lines:
          - authentication remote pre-share
          - authentication local pre-share
          - keyring local {{ IKEv2_keyring }}
          - match address local {{ public_ip }}
          - match identity remote address {{ remote_public_ip }} 255.255.255.255
        parents:
          - crypto ikev2 profile {{ IKEv2_profile }}
      notify: Save
    
    - name: Configure IPSEC transform set
      cisco.ios.ios_config:
        lines:
          - mode tunnel
        parents:
          - crypto ipsec transform-set {{ IPSEC_transform_set }} esp-aes 256 esp-sha512-hmac 
      notify: Save
    
    - name: Configure IPSEC profile
      cisco.ios.ios_config:
        lines:
          - set transform-set {{ IPSEC_transform_set }}
          - set ikev2-profile {{ IKEv2_profile }}
        parents:
          - crypto ipsec profile {{ IPSEC_profile }} 
      notify: Save
    
    # | Configure Tunnel interface |
    - name: Configure Tunnel0 interface
      cisco.ios.ios_config:
        lines:
          - tunnel source {{ tunnel_source }}
          - tunnel destination {{ remote_public_ip }}
          - tunnel protection ipsec profile {{ IPSEC_profile }}
          - no ip split-horizon eigrp {{ eigrp_as }}
          - no ipv6 split-horizon eigrp {{ eigrp_as }}
        parents:
          - interface Tunnel0
      notify: Save
    
    # | Configure EIGRP |
    - name: Configure EIGRP networks
      cisco.ios.ios_config:
        lines:
          - network {{ item.network }} {{ item.wildcard_mask }}
        parents:
          - router eigrp {{ eigrp_as }}
      loop: "{{ eigrp_network }}"
      notify: Save

    - name: Configure EIGRPv6 networks
      cisco.ios.ios_config:
        lines:
          - ipv6 eigrp {{ item.as }}
        parents:
          - interface {{ item.name }}
      loop: "{{ eigrp_v6_network }}"
      notify: 
        - Start Process
        - Save
    
    - name: Configure EIGRP passive interfaces
      cisco.ios.ios_config:
        lines:
          - passive-interface {{ item }}
        parents:
          - router eigrp {{ eigrp_as }}
      loop: "{{ eigrp_passive }}"
      notify: Save
    
    - name: Configure EIGRPv6 passive interfaces
      cisco.ios.ios_config:
        lines:
          - passive-interface {{ item }}
        parents:
          - ipv6 router eigrp {{ eigrp_as }}
      loop: "{{ eigrp_passive }}"
      notify: 
        - Start Process
        - Save
    
    - name: Configure EIGRP redistribute
      cisco.ios.ios_config:
        lines:
          - redistribute ospf {{ ospf_id }} metric 10000 1 255 1 1
        parents:
          - router eigrp {{ eigrp_as }}
      notify: Save
    
    - name: Configure EIGRPv6 redistribute
      cisco.ios.ios_config:
        lines:
          - redistribute ospf {{ ospf_id }} metric 10000 1 255 1 1 include-connected
        parents:
          - ipv6 router eigrp {{ eigrp_as }}
      notify: 
        - Start Process
        - Save
    
    # | Redistribute EIGRP into OSPF |
    - name: Configure OSPF redistribute
      cisco.ios.ios_config:
        lines:
          - redistribute eigrp {{ eigrp_as }} subnets
        parents:
          - router ospf {{ eigrp_as }}
      notify: Save
    
    - name: Configure OSPFv3 redistribute
      cisco.ios.ios_config:
        lines:
          - redistribute eigrp {{ eigrp_as }} include-connected
        parents:
          - ipv6 router ospf {{ eigrp_as }}
      notify: 
        - Start Process
        - Save
    
  handlers:
    # | Start EIGRPv6 |
    - name: Start EIGRPv6
      cisco.ios.ios_config:
        lines:
          - no shutdown
        parents:
          - ipv6 router eigrp {{ eigrp_as }}
      listen: Start Process

    # | Save settings |
    - name: Save IOS
      cisco.ios.ios_command:
        commands: "write mem"
      listen: Save

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