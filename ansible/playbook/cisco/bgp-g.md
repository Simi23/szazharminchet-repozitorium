---
title: Playbook: BGP (G)
description: Configure BGP through Ansible
published: true
date: 2025-03-01T13:52:19.908Z
tags: ansible, cisco
editor: markdown
dateCreated: 2025-03-01T13:52:19.908Z
---

# BGP

## Playbook

```yaml
---
- name: Configure BGP on ISP
  hosts: isp
  gather_facts: false
  tasks: 
    # | Configure BGP neigbors on ISP | 
    - name: Configure BGP neighbor on ISP
      cisco.ios.ios_bgp_global:
        config:
          as_number: "{{ item.as }}"
          neighbor: 
            - neighbor_address: "{{ item.ip }}"
              remote_as: "{{ item.nei_as }}"
      loop: "{{ bgp_nei }}"
      notify: Save

    # | Configure BGP networks on ISP | 
    - name: Configure BGP network on ISP
      cisco.ios.ios_bgp_global:
        config:
          as_number: "{{ item.as }}"
          networks:
            - address: "{{ item.network }}" 
              netmask: "{{ item.mask }}"
      loop: "{{ bgp_network }}"
      notify: Save

  handlers:
    # | Save IOS config |
    - name: Save IOS
      cisco.ios.ios_command:
        commands: "write mem"
      listen: Save


- name: Configure BGP on Border routers
  hosts: border_routers
  gather_facts: false
  tasks: 
    # | Configure BGP on Border routers | 
    - name: Configure BGP settings on Border routers
      cisco.ios.ios_bgp_global:
        config:
          as_number: "{{ bgp_as }}"
          neighbor: 
            - neighbor_address: "{{ item }}"
              remote_as: "{{ bgp_nei_as }}"
      loop: "{{ bgp_nei }}"
      notify: Save
  
  handlers:
    # | Save IOS config |
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
      
      bgp_as: 20
      bgp_nei: 
        - 1.1.1.1
        - 2001:DB8:1:1:1::1

    S2-BORDER.lego.dk:
      ansible_host: 172.20.0.14
      
      bgp_as: 30
      bgp_nei: 
        - 2.2.2.2
        - 2001:DB8:2:2:2::2

  vars:
    bgp_nei_as: 10
  
isp:
  hosts:
    ISP.lego.dk:
      ansible_host: 172.20.0.13
  
      bgp_nei:
        - nei_as: 20
          ip: 1.1.1.0
          as: 10
        - nei_as: 20
          ip: "2001:DB8:1:1:1::"
          as: 10
        - nei_as: 30
          ip: 2.2.2.1
          as: 10
        - nei_as: 30
          ip: 2001:DB8:2:2:2::1
          as: 10
      
      bgp_network:
        - network: 8.8.8.8
          mask: 255.255.255.255
          as: 10
        - network: 0.0.0.0
          mask: ""
          as: 10
```