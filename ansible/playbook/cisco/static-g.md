---
title: Playbook: Static IP routing (G)
description: Static IPv4, IPv6 routing configured with Ansible
published: true
date: 2025-03-01T14:06:31.216Z
tags: ansible, cisco
editor: markdown
dateCreated: 2025-03-01T13:48:47.518Z
---

# Playbook

```yaml
---
- name: Enable IPv6 routing on routers
  hosts: all
  gather_facts: false
  tasks:
    # | Enable IPv6 routing |
    - name: Enable IPv6 routing
      cisco.ios.ios_config:
        lines: "ipv6 unicast-routing"
      notify: Save
  
  handlers:
    # | Save IOS config |
    - name: Save IOS
      cisco.ios.ios_command:
        commands: "write mem"
      listen: Save

- name: Static ip routing on ISP
  hosts: isp
  gather_facts: false
  tasks:
    # | Configure default static IPv4, IPv6 route for BGP to advertise | 
    - name: Configure default static route for BGP to advertise
      cisco.ios.ios_static_routes:
        config:
          - address_families:
              - afi: ipv4
                routes: 
                  - dest: 0.0.0.0/0
                    next_hops:
                      - interface: Null0
              - afi: ipv6
                routes: 
                  - dest: ::/0
                    next_hops:
                      - interface: Null0
      notify: Save

  handlers:
    # | Save IOS config |
    - name: Save IOS
      cisco.ios.ios_command:
        commands: "write mem"
      listen: Save

```

# Inventory

```yaml
all:
  vars:
    ansible_user: ssh
    ansible_password: Passw0rd
    ansible_connection: ansible.netcommon.network_cli
    ansible_network_os: cisco.ios.ios

end_routers:
  hosts:
    S1.lego.dk:
      ansible_host: 172.20.0.11

    S2.lego.dk:
      ansible_host: 172.20.0.15

border_routers:
  hosts:
    S1-BORDER.lego.dk:
      ansible_host: 172.20.0.12
      
    S2-BORDER.lego.dk:
      ansible_host: 172.20.0.14

isp:
  hosts:
    ISP.lego.dk:
      ansible_host: 172.20.0.13
```