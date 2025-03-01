---
title: Playbook: IP settings (G)
description: IPv4, IPv6 settings, enabling interfaces using Ansible
published: true
date: 2025-03-01T13:43:00.658Z
tags: ansible, cisco
editor: markdown
dateCreated: 2025-03-01T13:43:00.658Z
---

# IP settings

## Playbook

```yaml
---
- name: Configure IP addresses
  hosts: all
  gather_facts: false
  tasks:
    # | Configre Cisco IOS interface addresses |
    - name: Configure IPv4, IPv6 addresses
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
    
    # | Enable Cisco IOS interfaces |
    - name: Enable interfaces
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
    - name: Save IOS
      cisco.ios.ios_command:
        commands: "write mem"
      listen: Save
```