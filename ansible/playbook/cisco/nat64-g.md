---
title: Playbook Collection: NAT64 (G)
description: NAT64 setup with Ansible
published: true
date: 2025-03-04T10:30:43.381Z
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
---
- name: Set interfaces
  hosts: routers
  gather_facts: false
  tasks:
    # | Set IP addresses |
    - name: Set IP addresses
      cisco.ios.ios_l3_interfaces: 
        config:
          - name: "{{ item.name }}"
            ipv4: 
              - address: "{{ item.ipv4 }}"
            ipv6:
              - address: "{{ item.ipv6 }}"
      loop: "{{ interfaces }}"
      loop_control:
        label: "{{ item.name }}"
      notify: Save
    
    # | Start interfaces |
    - name: Start interfaces
      cisco.ios.ios_interfaces: 
        config:
          - name: "{{ item.name }}"
            enabled: true
      loop: "{{ interfaces }}"
      loop_control:
        label: "{{ item.name }}"
      notify: Save

  handlers:
    # | Save IOS config |
    - name: Save IOS config
      cisco.ios.ios_command:
        commands:
          - command: "write memory"
      listen: Save

```
</details>

<details>
<summary>playbook_2_routing.yml</summary>
  
  ```yml
---
- name: Configure IP routing
  hosts: routers
  gather_facts: false
  tasks:
    # | Configure IPv4 routes |
    - name: Configure IPv4 routes
      cisco.ios.ios_config:
        lines:
          - ip route {{ item.network }} {{ item.destination }}
      loop: "{{ routesv4 }}"
      when: routesv4 is defined
      loop_control:
        label: "{{ item.network }}"
      notify: Save

    # | Configure IPv6 unicast-routing | 
    - name: Configure IPv6 unicast-routing
      cisco.ios.ios_config:
        lines: ipv6 unicast-routing
      notify: Save

    # | Configure IPv6 routes |
    - name: Configure IPv6 routes
      cisco.ios.ios_config:
        lines:
          - ipv6 route {{ item.network }} {{ item.destination }}
      loop: "{{ routesv6 }}"
      when: routesv6 is defined
      loop_control:
        label: "{{ item.network }}"
      notify: Save

  handlers:
    # | Save IOS config |
    - name: Save IOS config
      cisco.ios.ios_command:
        commands:
          - command: "write memory"
      listen: Save

```
</details>


<details>
<summary>playbook_3_nat64.yml</summary>

```yml
---
- name: NAT64 setup
  hosts: NAT64.lego.dk
  gather_facts: false
  tasks:
    # | Enable NAT64 |
    - name: Enable NAT64
      cisco.ios.ios_config:
        lines:
          - nat64 enable
        parents:
          - interface {{ item.name }}
      loop: "{{ nat64_interfaces }}"
      notify: Save
    
    # | Set up IPv6 ACL |
    - name: Set up IPv6 ACL 
      cisco.ios.ios_config:
        lines:
          - permit ipv6 {{ aclv6_permit }} any
        parents:
          - ipv6 access-list {{ aclv6_name }}
      notify: Save
    
    # | Set up NAT64 |
    - name: Set up NAT64
      cisco.ios.ios_config:
        lines:
          - nat64 v4 pool {{ nat64_name }} {{ nat64_start }} {{ nat64_end }}
          - nat64 v6v4 list {{ aclv6_name }} pool {{ nat64_name }} overload
      notify: Save

  handlers:
    # | Save IOS config |
    - name: Save IOS config
      cisco.ios.ios_command:
        commands:
          - command: "write memory"
      listen: Save
  
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
