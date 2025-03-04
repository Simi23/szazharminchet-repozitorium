---
title: Playbook Collection: NAT64
description: NAT64 setup with Ansible
published: true
date: 2025-03-04T10:06:05.787Z
tags: ansible, cisco
editor: markdown
dateCreated: 2025-03-04T10:06:05.787Z
---

# Introduction

For the manual configuration guide, check out [NAT64 configuration](/networking/nat64). The topology used is the same as in that guide.

# Playbooks

<details>
<summary>playbook_1_interfaces.yml</summary>
  
  ```yml
  - name: Setting up interfaces
    hosts: all
    tasks:
      - name: Setting IPv4 addresses
        cisco.ios.ios_l3_interfaces:
          config:
            - name: "{{ item.name }}"
              ipv4:
                - address: "{{ item.address4 }}"
        loop: "{{ interfaces }}"
        when: item.address4 is defined
        loop_control:
          label: "{{ item.name }}"

      - name: Setting IPv6 GUA addresses
        cisco.ios.ios_config:
          lines:
            - "ipv6 address {{ item.address6.gua }}"
          parents:
            - "interface {{ item.name }}"
        loop: "{{ interfaces }}"
        when: (item.address6 is defined) and (item.address6.gua is defined)
        loop_control:
          label: "{{ item.name }}"

      - name: Setting IPv6 LLA adresses
        cisco.ios.ios_config:
          lines:
            - ipv6 address {{ item.address6.lla }} link-local
          parents:
            - interface {{ item.name }}
        loop: "{{ interfaces }}"
        when: item.address6 is defined and item.address6.lla is defined
        loop_control:
          label: "{{ item.name }}"

      - name: Enabling interfaces
        cisco.ios.ios_interfaces:
          config:
            - name: "{{ item.name }}"
              enabled: true
        loop: "{{ interfaces }}"

```
</details>

<details>
<summary>playbook_2_routing.yml</summary>
  
  ```yml
  - name: Configuring routing
    hosts: all
    tasks:
      - name: Enabling IPv6 Unicast routing
        cisco.ios.ios_config:
          lines:
            - ipv6 unicast-routing
        when: ipv6_routing

      - name: Creating IPv4 static routes
        cisco.ios.ios_config:
          lines:
            - ip route {{ item.network }} {{ item.destination }}
        loop: "{{ routes4 }}"
        when: routes4 is defined
        loop_control:
          label: "{{ item.network }}"

      - name: Creating IPv6 static routes
        cisco.ios.ios_config:
          lines:
            - ipv6 route {{ item.network }} {{ item.destination }}
        loop: "{{ routes6 }}"
        when: routes6 is defined
        loop_control:
          label: "{{ item.network }}"


```
</details>


<details>
<summary>playbook_3_nat64.yml</summary>

```yml
  
  - name: Configuring NAT64
    hosts: NAT64.lego.dk
    tasks:
      - name: Creating IPv6 access list
        cisco.ios.ios_config:
          lines:
            - "permit ipv6 {{ item.src }} any"
          parents:
            - ipv6 access-list nat64acl
        loop: "{{ nat64.sources }}"
        loop_control:
          label: "{{ item.src }}"

      - name: Creating IPv4 translation pool
        cisco.ios.ios_config:
          lines:
            - nat64 v4 pool {{ nat64.destination.name }} {{ nat64.destination.start }} {{ nat64.destination.end }}

      - name: Enabling NAT64 on interfaces
        cisco.ios.ios_config:
          lines:
            - nat64 enable
          parents:
            - interface {{ item }}
        loop: "{{ nat64.interfaces }}"
        loop_control:
          label: "{{ item }}"

      - name: Creating NAT64 mapping
        cisco.ios.ios_config:
          lines:
            - nat64 v6v4 list nat64acl pool {{ nat64.destination.name }} overload

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
    hosts:
      R-CE.lego.dk:
        ansible_host: 10.0.0.1
        ipv6_routing: true
        interfaces:
          - name: GigabitEthernet0/0
            address6:
              gua: 2001:DB8:10::1/64
          - name: GigabitEthernet0/1
            address6:
              gua: 2001:DB8:20::DEF/64
              lla: FE80::DEF
        routes6:
          - network: ::/0
            destination: 2001:DB8:10::64
      NAT64.lego.dk:
        ansible_host: 10.0.0.2
        ipv6_routing: true
        interfaces:
          - name: GigabitEthernet0/0
            address6:
              gua: 2001:DB8:10::64/64
          - name: GigabitEthernet0/1
            address4: 10.0.0.64/24
        routes4:
          - network: 8.8.8.0 255.255.255.0
            destination: 10.0.0.1
        routes6:
          - network: 2001:DB8:20::/64
            destination: 2001:DB8:10::1
        nat64:
          sources:
            - src: 2001:DB8:20::/64
          destination:
            name: POOL
            start: 10.0.0.100
            end: 10.0.0.100
          interfaces:
            - GigabitEthernet0/0
            - GigabitEthernet0/1
      R-WEB.lego.dk:
        ansible_host: 10.0.0.3
        ipv6_routing: false
        interfaces:
          - name: GigabitEthernet0/0
            address4: 8.8.8.254/24
          - name: GigabitEthernet0/1
            address4: 10.0.0.1/24
        routes4:
          - network: 0.0.0.0 0.0.0.0
            destination: 10.0.0.64


```

</details>
