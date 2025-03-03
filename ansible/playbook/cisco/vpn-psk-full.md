---
title: Playbook Collection: PSK IKEv2 VPN Setup
description: IPv4, IPv6 settings, routing protocols, IKEv2 PSK Authentication
published: true
date: 2025-03-03T08:48:47.698Z
tags: ansible, cisco
editor: markdown
dateCreated: 2025-03-03T08:48:47.698Z
---

# Playbooks

<details>
<summary>playbook_1_interfaces.yml</summary>
  
  ```yml
  ---
  - name: Configuring interfaces
    hosts: all
    tasks:
      - name: Setting IPv4 addresses
        cisco.ios.ios_l3_interfaces:
          config:
            - name: "{{ item.name }}"
              ipv4:
                - address: "{{ item.address }}"
        loop: "{{ interfaces }}"
        loop_control:
          label: "{{ item.name }}"

      - name: Setting IPv6 addresses
        cisco.ios.ios_l3_interfaces:
          config:
            - name: "{{ item.name }}"
              ipv6: "{{ item.address6 }}"
        loop: "{{ interfaces }}"
        when: item.address6 is defined
        loop_control:
          label: "{{ item.name }}"

      - name: Enabling interfaces
        cisco.ios.ios_interfaces:
          config:
            - name: "{{ item.name }}"
              enabled: true
        loop: "{{ interfaces }}"
        loop_control:
          label: "{{ item.name }}"

      - name: Saving config
        cisco.ios.ios_config:
          save_when: modified

```
</details>

<details>
<summary>playbook_2_routing.yml</summary>
  
  ```yml
  ---
  - name: Configuring routing
    hosts: all
    tasks:
      - name: Enabling IPv6 routing
        cisco.ios.ios_config:
          lines:
            - ipv6 unicast-routing

      - name: Configuring static routes # noqa args[module]
        cisco.ios.ios_static_routes:
          config:
            - address_families: "{{ routing.static }}"
        when: routing.static is defined

      - name: Configuring EIGRP
        cisco.ios.ios_config:
          lines: "{{ item.config_lines }}"
          parents:
            - "router eigrp {{ item.asn }}"
        when: routing.eigrp is defined
        loop: "{{ routing.eigrp }}"
        loop_control:
          label: "ASN {{ item.asn }}"

      - name: Configuring EIGRPv6 processes
        cisco.ios.ios_config:
          lines:
            - "ipv6 eigrp {{ item.asn }}"
          parents:
            - interface {{ item.interface }}
        when: routing.eigrpv6 is defined
        loop: "{{ routing.eigrpv6 }}"
        loop_control:
          label: "ASN {{ item.asn }} - {{ item.interface }}"

      - name: Configuring EIGRPv6 interfaces
        cisco.ios.ios_config:
          lines:
            - "ipv6 eigrp {{ item.asn }}"
          parents:
            - interface {{ item.interface }}
        when: routing.eigrpv6 is defined
        loop: "{{ routing.eigrpv6 }}"
        loop_control:
          label: "ASN {{ item.asn }} - {{ item.interface }}"

      - name: Configuring OSPF # noqa args[module]
        cisco.ios.ios_ospfv2:
          config:
            processes: "{{ routing.ospf }}"
        when: routing.ospf is defined

      - name: Configuring OSPF (Other)
        cisco.ios.ios_config:
          lines: "{{ item.lines }}"
          parents:
            - "router ospf {{ item.process_id }}"
        loop: "{{ routing.ospf_other }}"
        when: routing.ospf_other is defined
        loop_control:
          label: "PID {{ item.process_id }}"

      - name: Configuring BGP
        cisco.ios.ios_config:
          lines: "{{ routing.bgp.global.config_lines }}"
          parents:
            - "router bgp {{ routing.bgp.global.as_number }}"
        when: routing.bgp is defined

      - name: Saving config
        cisco.ios.ios_config:
          save_when: modified

```
</details>

<details>
  <summary>playbook_3_vpn.yml</summary>
  
  ```yml
    ---
  - name: Configuring VPN
    hosts: vpn_endpoints
    tasks:
      - name: Creating IKEv2 keyrings
        cisco.ios.ios_config:
          lines:
            - address {{ item.peer.address }}
            - pre-shared-key {{ vpn.psk }}
          parents:
            - crypto ikev2 keyring KEYRING
            - peer {{ item.peer.name }}
        loop: "{{ vpn_tunnels }}"
        loop_control:
          label: "{{ item.local.address }} <=> {{ item.peer.name }}"

      - name: Creating certificate maps
        cisco.ios.ios_config:
          lines:
            - "{{ item.certificate.map }}"
          parents:
            - "crypto pki certificate map CMAP 10"
        loop: "{{ vpn_tunnels }}"
        loop_control:
          label: "{{ item.peer.identity }}"

      - name: Creating IKEv2 Proposals
        cisco.ios.ios_config:
          lines: "{{ item.ciphers }}"
          parents:
            - crypto ikev2 proposal {{ item.name }}
        loop: "{{ vpn.proposals }}"
        loop_control:
          label: "{{ item.name }}"

      - name: Creating IKEv2 Policies
        cisco.ios.ios_config:
          lines: "{{ item.proposals }}"
          parents:
            - crypto ikev2 policy {{ item.name }}
        loop: "{{ vpn.policies }}"
        loop_control:
          label: "{{ item.name }}"

      - name: Creating IKEv2 PSK profiles
        cisco.ios.ios_config:
          lines:
            - match address local {{ vpn_tunnels[item.tunnel_id].local.address }}
            - match identity remote address {{ vpn_tunnels[item.tunnel_id].peer.address }} 255.255.255.255
            - authentication remote pre-share
            - authentication local pre-share
            - keyring local KEYRING
          parents:
            - crypto ikev2 profile {{ item.name }}
        loop: "{{ vpn.ikev2_profiles }}"
        when: item.type == "psk"
        loop_control:
          label: "{{ item.name }}"

      - name: Creating IKEv2 Certificate profiles
        cisco.ios.ios_config:
          lines:
            - match identity remote fqdn {{ vpn_tunnels[item.tunnel_id].peer.identity }}
            - match certificate CMAP
            - identity local fqdn {{ vpn_tunnels[item.tunnel_id].local.identity }}
            - authentication remote rsa-sig
            - authentication local rsa-sig
            - pki trustpoint {{ ca_name }}
          parents:
            - crypto ikev2 profile {{ item.name }}
        loop: "{{ vpn.ikev2_profiles }}"
        when: item.type == "cert"
        loop_control:
          label: "{{ item.name }}"

      - name: Creating IPsec transform sets
        cisco.ios.ios_config:
          lines:
            - mode {{ item.mode }}
          parents:
            - crypto ipsec transform-set {{ item.name }} {{ item.encryption }} {{ item.authentication }}
        loop: "{{ vpn.transform_sets }}"
        loop_control:
          label: "{{ item.name }}"

      - name: Creating IPsec profiles
        cisco.ios.ios_config:
          lines:
            - set transform-set {{ item.transform_set }}
            - set ikev2-profile {{ item.ikev2_profile }}
          parents:
            - crypto ipsec profile {{ item.name }}
        loop: "{{ vpn.ipsec_profiles }}"
        loop_control:
          label: "{{ item.name }}"

      - name: Setting up tunnel interfaces
        cisco.ios.ios_config:
          lines:
            - tunnel source {{ item.local.source }}
            - tunnel destination {{ item.peer.address }}
            - tunnel protection ipsec profile {{ item.profile }}
          parents:
            - interface {{ item.local.tunnel }}
        loop: "{{ vpn_tunnels }}"
        loop_control:
          label: "{{ item.local.tunnel }}"

      - name: Saving config
        cisco.ios.ios_config:
          save_when: modified

  ```
</details>

# Inventory

<details>
  <summary>inventory.yml</summary>
	
  ```yml
  all:
    vars:
      ansible_connection: ansible.netcommon.network_cli
      ansible_network_os: cisco.ios.ios
      ansible_user: ansible
      ansible_password: ansible
      ssh_type: paramiko

  vpn_endpoints:
    hosts:
      s1-border.lego.dk:
      s2-border.lego.dk:
    vars:
      vpn:
        psk: Passw0rd
        proposals:
          - name: PSK-PROP
            ciphers:
              - encryption aes-gcm-256
              - prf sha512
              - group 21
        policies:
          - name: PSK-POLICY
            proposals:
              - proposal PSK-PROP
        ikev2_profiles:
          - name: PSK-PROFILE
            type: psk
            tunnel_id: 0
        transform_sets:
          - name: TSET
            encryption: esp-aes 256
            authentication: esp-sha512-hmac
            mode: tunnel
        ipsec_profiles:
          - name: PSK-PROFILE
            transform_set: TSET
            ikev2_profile: PSK-PROFILE

  site_s1:
    hosts:
      s1-border.lego.dk:
        ansible_host: 10.0.0.11
        interfaces:
          - name: GigabitEthernet0/0
            address: 31.46.52.82/30
          - name: GigabitEthernet0/1
            address: 10.1.200.1/30
          - name: Tunnel1
            address: 10.200.0.1/30
        routing:
          eigrp:
            - asn: 10
              config_lines:
                - network 10.1.200.0 0.0.0.3
                - redistribute eigrp 200
            - asn: 200
              config_lines:
                - network 10.200.0.0 0.0.0.3
                - redistribute eigrp 10
          bgp:
            global:
              as_number: 10
              config_lines:
                - neighbor 31.46.52.81 remote-as 30
                - neighbor 31.46.52.81 description ISP
        vpn_tunnels:
          - profile: PSK-PROFILE
            peer:
              name: s2-border.lego.dk
              address: 84.72.45.26
            local:
              address: 31.46.52.82
              source: GigabitEthernet0/0
              tunnel: Tunnel1

      s1.lego.dk:
        ansible_host: 10.0.0.12
        interfaces:
          - name: GigabitEthernet0/1
            address: 10.1.200.2/30
          - name: Loopback0
            address: 10.1.0.254/24
        routing:
          eigrp:
            - asn: 10
              config_lines:
                - network 10.1.200.0 0.0.0.3
                - network 10.1.0.0 0.0.0.255

  site_s2:
    hosts:
      s2-border.lego.dk:
        ansible_host: 10.0.0.21
        interfaces:
          - name: GigabitEthernet0/0
            address: 84.72.45.26/30
          - name: GigabitEthernet0/1
            address: 10.2.200.1/30
          - name: Tunnel1
            address: 10.200.0.2/30
        routing:
          ospf:
            - process_id: 20
              network:
                - address: 10.2.200.0
                  wildcard_bits: 0.0.0.3
                  area: 0
          ospf_other:
            - process_id: 20
              lines:
                - redistribute eigrp 200 subnets
          eigrp:
            - asn: 200
              config_lines:
                - network 10.200.0.0 0.0.0.3
                - redistribute ospf 20 metric 1000000 1 255 1 1500
          bgp:
            global:
              as_number: 20
              config_lines:
                - neighbor 84.72.45.25 remote-as 30
                - neighbor 84.72.45.25 description ISP
        vpn_tunnels:
          - profile: PSK-PROFILE
            peer:
              name: s1-border.lego.dk
              address: 31.46.52.82
            local:
              address: 84.72.45.26
              source: GigabitEthernet0/0
              tunnel: Tunnel1

      s2.lego.dk:
        ansible_host: 10.0.0.22
        interfaces:
          - name: GigabitEthernet0/1
            address: 10.2.200.2/30
          - name: Loopback0
            address: 10.2.0.254/24
        routing:
          ospf:
            - process_id: 20
              network:
                - address: 10.2.200.0
                  wildcard_bits: 0.0.0.3
                  area: 0
                - address: 10.2.0.0
                  wildcard_bits: 0.0.0.255
                  area: 0

  site_isp:
    hosts:
      isp.lego.dk:
        ansible_host: 10.0.0.31
        interfaces:
          - name: GigabitEthernet0/0
            address: 31.46.52.81/30
          - name: GigabitEthernet0/1
            address: 84.72.45.25/30
          - name: Loopback0
            address: 8.8.8.8/32
        routing:
          static:
            - afi: ipv4
              routes:
                - dest: 0.0.0.0/0
                  next_hops:
                    - interface: Null0
          bgp:
            global:
              as_number: 30
              config_lines:
                - neighbor 31.46.52.82 remote-as 10
                - neighbor 31.46.52.82 description S1-BORDER
                - neighbor 84.72.45.26 remote-as 20
                - neighbor 84.72.45.26 description S2-BORDER
                - network 8.8.8.8 mask 255.255.255.255
                - network 0.0.0.0


```

</details>
