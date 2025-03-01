---
title: Playbook: IP settings (G)
description: IPv4, IPv6 settings, enabling interfaces using Ansible
published: true
date: 2025-03-01T14:05:50.407Z
tags: ansible, cisco
editor: markdown
dateCreated: 2025-03-01T13:43:00.658Z
---

# Playbook

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
      
      interfaces: 
        - name: GigabitEthernet0/1
          ipv4: 10.1.255.0/31
          ipv6: 2001:db8:1:255::/64
        - name: Loopback0
          ipv4: 10.255.255.1/32
          ipv6: 2001:db8:255:255::1/128
    
    S2.lego.dk:
      ansible_host: 172.20.0.15
      
      interfaces: 
        - name: GigabitEthernet0/1
          ipv4: 10.2.255.0/31
          ipv6: 2001:db8:2:255::/64
        - name: Loopback0
          ipv4: 10.255.255.4/32
          ipv6: 2001:db8:255:255::1/128

border_routers:
  hosts:
    S1-BORDER.lego.dk:
      ansible_host: 172.20.0.12

      interfaces: 
        - name: GigabitEthernet0/0
          ipv4: 1.1.1.0/31
          ipv6: 2001:db8:1:1:1::/64
        - name: GigabitEthernet0/1
          ipv4: 10.1.255.1/31
          ipv6: 2001:db8:1:255::1/64
        - name: Tunnel0
          ipv4: 10.0.0.1/24
          ipv6: 2001:db8:10::1/64
        - name: Loopback0
          ipv4: 10.255.255.2/32
          ipv6: 2001:db8:255:255::2/128

    S2-BORDER.lego.dk:
      ansible_host: 172.20.0.14

      interfaces:
        - name: GigabitEthernet0/0
          ipv4: 2.2.2.1/30
          ipv6: 2001:db8:2:2:2::1/64
        - name: GigabitEthernet0/1
          ipv4: 10.2.255.1/31
          ipv6: 2001:db8:2:255::1/64
        - name: Tunnel0
          ipv4: 10.0.0.2/24
          ipv6: 2001:db8:10::2/64
        - name: Loopback0
          ipv4: 10.255.255.3/32
          ipv6: 2001:db8:255:255::3/128
  
isp:
  hosts:
    ISP.lego.dk:
      ansible_host: 172.20.0.13

      interfaces: 
        - name: GigabitEthernet0/0
          ipv4: 1.1.1.1/31
          ipv6: 2001:db8:1:1:1::1/64
        - name: GigabitEthernet0/1
          ipv4: 2.2.2.2/30
          ipv6: 2001:db8:2:2:2::2/64
        - name: Loopback0
          ipv4: 8.8.8.8/32
          ipv6: 2001:db8:8:8:8::8/128
  
```