---
title: Playbook Collection: NAT64 (G)
description: NAT64 setup with Ansible
published: true
date: 2025-03-04T10:29:53.164Z
tags: ansible, cisco
editor: markdown
dateCreated: 2025-03-04T10:29:53.164Z
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
    ansible_user: ssh
    ansible_password: Passw0rd
    ansible_connection: ansible.netcommon.network_cli
    ansible_network_os: cisco.ios.ios

routers:
  hosts:
    R-CE.lego.dk:
      ansible_host: 172.20.0.13

      interfaces:
        - name: GigabitEthernet0/0
          ipv4: 1.1.1.1/31
          ipv6: 2001:db8:20::254/64
        - name: GigabitEthernet0/1
          ipv4: 1.1.1.2/31
          ipv6: 2001:db8:10::1/64
      
      routesv6:
        - network: ::/0
          destination: 2001:DB8:10::2
    
    NAT64.lego.dk:
      ansible_host: 172.20.0.14

      interfaces:
        - name: GigabitEthernet0/2
          ipv4: 10.1.1.1/24
          ipv6: 2001:db8::fe80/128
        - name: GigabitEthernet0/1
          ipv4: 1.1.1.3/31
          ipv6: 2001:db8:10::2/64

      routesv4:
        - network: 8.8.8.0 255.255.255.0
          destination: 10.1.1.2
      
      routesv6:
        - network: 2001:DB8:20::/64
          destination: 2001:DB8:10::1

      
      aclv6_name: "NAT64ACL"
      aclv6_permit: "2001:DB8::/32"

      nat64_interfaces:
        - name: GigabitEthernet0/1
        - name: GigabitEthernet0/2
      nat64_name: IPV4POOL
      nat64_start: 10.1.1.100
      nat64_end: 10.1.1.200
    
    R-WEB.lego.dk:
      ansible_host: 172.20.0.15

      interfaces:
        - name: GigabitEthernet0/2
          ipv4: 10.1.1.2/24
          ipv6: 2001:db8::fe81/128
        - name: GigabitEthernet0/0
          ipv4: 8.8.8.1/24
          ipv6: 2001:db8::fe82/128

      routesv4:
        - network: 0.0.0.0 0.0.0.0
          destination: 10.1.1.1
```

</details>
