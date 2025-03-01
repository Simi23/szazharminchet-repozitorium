---
title: Playbook: OSPF (G)
description: Configure OSPFv2, OSPFv3 using Ansible
published: true
date: 2025-03-01T13:57:16.070Z
tags: ansible, cisco
editor: markdown
dateCreated: 2025-03-01T13:57:16.070Z
---

# OSPF

## Playbook

```yaml
---
- name: Configure OSPF routing on border routers
  hosts: border_routers
  gather_facts: false
  tasks:
    # | Completly configure OSPFv2 on border routers. |
    - name: Configure single area OSPFv2 on border routers (Default information)
      cisco.ios.ios_ospfv2:
        config:
          processes:
            - process_id: "{{ ospf_id }}"
              default_information:
                originate: true

    - name: Configure single area OSPFv2 on border routers (Networks)
      cisco.ios.ios_ospfv2:
        config:
          processes:
            - process_id: "{{ ospf_id }}"
              network:
                - address: "{{ item.network}}"
                  wildcard_bits: "{{ item.wildcard_mask}}"
                  area: "{{ item.area}}"
      loop: "{{ ospf_nei }}"
      loop_control:
        label: "{{ item.network }}"
      notify: Save

    - name: Configure single area OSPFv2 on border routers (Passive interfaces)
      cisco.ios.ios_ospfv2:
        config:
          processes:
            - process_id: "{{ ospf_id }}"
              passive_interfaces:
                interface:
                  name: "{{ item }}"
                  set_interface: true
      loop: "{{ ospf_passive }}"
      notify: Save
    
    # | Completly configure OSPFv3 on border routers. |
    - name: Configure single area OSPFv3 on border routers (Networks)
      cisco.ios.ios_ospf_interfaces:
        config:
          - name: "{{ item.name }}"
            address_family:
              - afi: ipv6
                process:
                  id: "{{ item.process }}"
                  area_id: "{{ item.area }}"
                adjacency: true
      loop: "{{ ospfv3_nei }}"
      loop_control:
        label: "{{ item.name }}"
      notify: Save

  handlers:
    # | Save IOS config |
    - name: Save IOS
      cisco.ios.ios_command:
        commands: "write mem"
      listen: Save
              

- name: Configure OSPF routing on site routers
  hosts: end_routers
  gather_facts: false
  tasks:    
    # | Completly configure OSPFv2 on site routers. |
    - name: Configure single area OSPFv2 on site routers (Networks)
      cisco.ios.ios_ospfv2:
        config:
          processes:
            - process_id: "{{ ospf_id }}"
              network:
                - address: "{{ item.network}}"
                  wildcard_bits: "{{ item.wildcard_mask}}"
                  area: "{{ item.area}}"
      loop: "{{ ospf_nei }}"
      loop_control:
        label: "{{ item.network }}"
      notify: Save


    - name: Configure single area OSPFv2 on site routers (Passive interface)
      cisco.ios.ios_ospfv2:
        config:
          processes:
            - process_id: "{{ ospf_id }}"
              passive_interfaces:
                interface:
                  name: "{{ item }}"
                  set_interface: true
      loop: "{{ ospf_passive }}"
      notify: Save
    
    # | Completly configure OSPFv3 on site routers. |
    - name: Configure single area OSPFv3 on end routers (Networks)
      cisco.ios.ios_ospf_interfaces:
        config:
          - name: "{{ item.name }}"
            address_family:
              - afi: ipv6
                process:
                  id: "{{ item.process }}"
                  area_id: "{{ item.area }}"
                adjacency: true
      loop: "{{ ospfv3_nei }}"
      loop_control:
        label: "{{ item.name }}"
      notify: Save
  
  handlers:
    # | Save IOS config |
    - name: Save IOS
      cisco.ios.ios_command:
        commands: "write mem"
      listen: Save

- name: Redistribute BGP into OSPF
  hosts: border_routers
  gather_facts: false
  tasks:
    # | Check redistribute for BGP in OSPF |
    - name: Check if BGP redistributed into OSPF
      cisco.ios.ios_command:
        commands: "show run | include redistribute bgp"
      register: bgp_red
      changed_when: '"bgp" not in bgp_red.stdout[0]'
      notify:
        - Redistribute
        - Save

  handlers:
    # | Add redistribute for BGP into OSPF |
    - name: Redistribute BGP into OSPF
      cisco.ios.ios_command:
        commands:
          - conf t
          - router ospf {{ ospf_id }}
          - redistribute bgp {{ bgp_as }} subnets
          - ipv6 router ospf {{ ospf_id }}
          - redistribute bgp {{ bgp_as }}
      listen: Redistribute

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

end_routers:
  hosts:
    S1.lego.dk:
      ansible_host: 172.20.0.11
      ospf_nei:
        - network: 10.1.255.0
          wildcard_mask: 0.0.0.1
          area: 0
        - network: 10.255.255.1
          wildcard_mask: 0.0.0.0
          area: 0
    
    S2.lego.dk:
      ansible_host: 172.20.0.15
      ospf_nei:
        - network: 10.2.255.0
          wildcard_mask: 0.0.0.1
          area: 0
        - network: 10.255.255.4
          wildcard_mask: 0.0.0.0
          area: 0

  vars:
    ospf_id: 10
    ospf_passive:
      - GigabitEthernet0/0
      - GigabitEthernet0/2
      - Loopback0
    
    ospfv3_nei:
      - name: GigabitEthernet0/1
        process: 10
        area: 0
      - name: Loopback0
        process: 10
        area: 0

border_routers:
  hosts:
    S1-BORDER.lego.dk:
      ansible_host: 172.20.0.12
      ospf_nei:
        - network: 10.1.255.0
          wildcard_mask: 0.0.0.1
          area: 0
        - network: 10.255.255.2
          wildcard_mask: 0.0.0.0
          area: 0

    S2-BORDER.lego.dk:
      ansible_host: 172.20.0.14
      ospf_nei:
        - network: 10.2.255.0
          wildcard_mask: 0.0.0.1
          area: 0
        - network: 10.255.255.3
          wildcard_mask: 0.0.0.0
          area: 0
      
  vars:
    ospf_id: 10
    ospf_passive:
      - GigabitEthernet0/0
      - GigabitEthernet0/2
      - Loopback0
      - Tunnel0

    ospfv3_nei:
      - name: GigabitEthernet0/1
        process: 10
        area: 0
      - name: Loopback0
        process: 10
        area: 0
```