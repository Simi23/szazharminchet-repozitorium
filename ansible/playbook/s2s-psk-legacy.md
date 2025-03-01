---
title: Playbook: Site-to-site VPN with PSK
description: Site-to-site VPN Ansible playbook for RRAS - Strongswan (legacy)
published: true
date: 2025-03-01T14:05:11.097Z
tags: linux, windows, ansible
editor: markdown
dateCreated: 2025-02-24T08:02:05.278Z
---

# Playbook

```yaml
---
- name: Install RRAS
  hosts: WIN-SRV-AD.TEST.DK
  gather_facts: false
  tasks:
    - name: Install RRAS
      ansible.windows.win_feature:
        name:
          - RemoteAccess
          - Routing
        state: present
        include_management_tools: true

    - name: Check VPN service installation (1/2)
      ansible.windows.win_shell: 'Get-RemoteAccess'
      register: vpn
      changed_when: '"RoutingStatus      : Installed" not in vpn.stdout'
      notify: Configure VPN RoutingOnly
    
    - name: Check VPN service installation (2/2)
      ansible.windows.win_shell: 'Get-RemoteAccess'
      register: vpn
      changed_when: '"VpnS2SStatus       : Installed" not in vpn.stdout'
      notify: Configure VPN S2S
    
    - name: Run handlers for Services
      ansible.builtin.meta: flush_handlers

    - name: Check VPN status
      ansible.windows.win_shell: 'Get-VpnS2SInterface | Select-Object -ExpandProperty Name'
      register: vpns2s
      changed_when: '"LIN-RTR-PSK" not in vpns2s.stdout'
      notify: Configure S2S Interface

  handlers:    
    - name: Install VPN RoutingOnly
      win_shell: >
        Install-RemoteAccess -Legacy -VpnType RoutingOnly
      listen: Configure VPN RoutingOnly
    
    - name: Install VPN VpnS2S
      win_shell: >
        Install-RemoteAccess -Legacy -VpnType VpnS2S
      listen: Configure VPN S2S

    - name: Add new Demand-dial interface 
      win_shell: >
        Add-VpnS2SInterface -Name "LIN-RTR-PSK" 
        -Destination 20.0.0.2 
        -AuthenticationMethod PSKOnly 
        -SharedSecret "Passw0rd" 
        -Persistent 
        -IPv4Subnet "10.10.0.0/24:10" 
        -Protocol IKEv2 
      listen: Configure S2S Interface

- name: S2S Strongswan PSK - Legacy
  hosts: lin_rtr
  gather_facts: false
  tasks:
    - name: Install Strongswan packages
      ansible.builtin.apt:
        name: 
          - strongswan
          - strongswan-starter
          - libcharon-extra-plugins
        state: present

    - name: Copy ipsec.conf (1/2)
      ansible.builtin.template:
        src: templates/ipsec.conf.s2spsk.j2
        dest: /etc/ipsec.conf
        owner: root
        group: root
        mode: "0644"
      notify: 
        - Restart strongswan
    
    - name: Copy ipsec.secrets (2/2)
      ansible.builtin.template:
        src: templates/ipsec.secrets.s2spsk.j2
        dest: /etc/ipsec.secrets
        owner: root
        group: root
        mode: "0644"
      notify:
        - Restart strongswan
  
  handlers:
    - name: Restart strongswan
      ansible.builtin.service:
        name: strongswan-starter
        state: restarted
```

# Inventory

```yaml
all:
  vars:
    s2s:
      debian_subnet: 10.10.0.0/24
      debian_public_ipv4: 20.0.0.2
      windows_subnet: 10.20.0.0/24
      windows_public_ipv4: 20.0.0.3
      tunnel_subnet: 10.255.255.0/24
      PSK: "Passw0rd"
      winid: '"CN=WIN-SRV-AD.test.dk"'

debian_servers:
  hosts:
    lin_rtr:
      ansible_host: 10.10.0.254

windows_servers:
  hosts:
    WIN-SRV-AD.TEST.DK:
      ansible_host: WIN-SRV-AD.TEST.DK
      ipv4_add: 10.20.0.254
      ansible_connection: winrm
      ansible_winrm_transport: ntlm
      ansible_port: 5985
      ansible_user: Administrator@TEST.DK
      ansible_become_user: TEST\Administrator
      ansible_become_method: runas
      ansible_password: Passw0rd
      ansible_become_password: Passw0rd  
```

# Templates

<kbd>ipsec.conf.s2spsk.j2</kbd>

```c
conn rras-psk
    # General settings
    authby=psk
    keyexchange=ikev2
    ike=aes256-sha2_256-modp1024
    esp=aes256-sha2_256
    auto=start

    # Debian side settings
    left= {{ s2s.debian_public_ipv4 }}
    leftsubnet= {{ s2s.debian_subnet }}

    # Windows side settings
    right= {{ s2s.windows_public_ipv4 }}
    rightsubnet= {{ s2s.windows_subnet }}
    rightsourceip= {{ s2s.tunnel_subnet }}
```

<kbd>ipsec.secrets.s2spsk.j2</kbd>

```c
{{ s2s.debian_public_ipv4 }} {{ s2s.windows_public_ipv4 }} : PSK {{ s2s.PSK }}
```