---
title: Playbook: Site-to-site VPN with PSK
description: Site-to-site VPN Ansible playbook for RRAS - Strongswan (swanctl)
published: true
date: 2025-02-21T06:43:07.425Z
tags: linux, windows, ansible
editor: markdown
dateCreated: 2025-02-21T06:43:07.425Z
---

# Playbook

```yaml
---
- name: RRAS Setup
  hosts: WIN-SRV.TEST.DK
  become: true
  tasks:
    - name: Installing RRAS
      ansible.windows.win_feature:
        name:
          - DirectAccess-VPN
          - Routing
        include_management_tools: true

    - name: Checking S2S Installation
      ansible.windows.win_shell: >
        (Get-RemoteAccess | Where-Object { $_.VpnS2SStatus -Like "Installed" }) -ne $null
      register: s2s
      changed_when: '"True" not in s2s.stdout'
      notify: install_s2s

    - name: Checking Routing Installation
      ansible.windows.win_shell: >
        (Get-RemoteAccess | Where-Object { $_.RoutingStatus -Like "Installed" }) -ne $null
      register: routing
      changed_when: '"True" not in routing.stdout'
      notify: install_routing

    - name: Installing needed packages
      ansible.builtin.meta: flush_handlers

    - name: Checking S2S Interface
      ansible.windows.win_shell: >
        (Get-VpnS2SInterface | Where-Object { $_.Name -Like "rtr.lego.dk" }) -ne $null
      register: interface
      changed_when: '"True" not in interface.stdout'
      notify: add_interface

  handlers:
    - name: Install S2S
      ansible.windows.win_shell: >
        Install-RemoteAccess
        -Legacy
        -VpnType VpnS2S
      listen: install_s2s

    - name: Install Routing
      ansible.windows.win_shell: >
        Install-RemoteAccess
        -Legacy
        -VpnType RoutingOnly
      listen: install_routing

    - name: Adding S2S Tunnel
      listen: add_interface
      ansible.windows.win_shell: >
        Add-VpnS2SInterface
        -Name "rtr.lego.dk"
        -Destination 20.0.0.2
        -Protocol IKEv2
        -AuthenticationMethod PSKOnly
        -SharedSecret Passw0rd
        -Persistent
        -IPv4Subnet 10.10.0.0/24:10

- name: Strongswan Setup
  hosts: RTR.LEGO.DK
  tasks:
    - name: Install Strongswan
      ansible.builtin.apt:
        name:
          - charon-systemd
          - strongswan-swanctl
    - name: Configure Strongswan
      ansible.builtin.template:
        src: "templates/strongswan-s2s-psk.j2"
        dest: "/etc/swanctl/swanctl.conf"
        mode: "644"
      notify: strongswan_config
  handlers:
    - name: Load Strongswan configuration
      ansible.builtin.shell: >
        swanctl --load-all
      changed_when: true
      listen: strongswan_config

```

# Inventory

```yaml
windows:
  hosts:
    WIN-SRV.TEST.DK:
      ansible_host: WIN-SRV.TEST.DK
      ansible_port: 5985
      ansible_user: Administrator@TEST.DK
      ansible_password: Passw0rd
      ansible_connection: winrm
      ansible_winrm_transport: ntlm
      ansible_become_method: runas
      ansible_become_user: TEST\Administrator
      ansible_become_password: Passw0rd

linux:
  vars:
      ansible_user: root
      ansible_password: Passw0rd
      ansible_python_interpreter: auto_silent
  hosts:
    RTR.LEGO.DK:
			ansible_host: rtr.lego.dk
```

# Templates

<kbd>templates/strongswan-s2s-psk.j2</kbd>

```c
connections {
    s2s {
        local_addrs  = 20.0.0.2
        remote_addrs = 20.0.0.3
        proposals = aes256gcm16-prfsha384-modp1024
        pools = pool1

        local {
            auth = psk
            id = 20.0.0.2
        }
        remote {
            auth = psk
            id = 20.0.0.3
        }
        children {
            net {
                local_ts = 10.10.0.0/24
                remote_ts = 10.20.0.0/24
                start_action = start
            }
        }
        version = 2
    }
}

pools {
    pool1 {
        addrs = 192.0.2.0/24
    }
}

secrets {
    ike1 {
        id = 20.0.0.2
        id = 20.0.0.3
        secret = "Passw0rd"
    }
}

include conf.d/*.conf
```