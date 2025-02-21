---
title: Playbook: Site-to-site VPN with Certificate
description: Site-to-site VPN Ansible playbook for RRAS - Strongswan (swanctl)
published: true
date: 2025-02-21T06:46:56.027Z
tags: linux, windows, ansible
editor: markdown
dateCreated: 2025-02-21T06:46:56.027Z
---

> This guide assumes you already have certificates set up. On the Windows host they have to be already installed, and for the linux host, they have to be on the Ansible host.
{.is-info}

# Playbook

```yaml
---
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
        src: "templates/strongswan-s2s-cert.j2"
        dest: "/etc/swanctl/swanctl.conf"
        mode: "644"
      notify: strongswan_config

    - name: Copying CA Certificate
      ansible.builtin.copy:
        src: "certs/ca.pem"
        dest: "/etc/swanctl/x509ca/ca.pem"
        mode: "644"

    - name: Copying VPN Certificate
      ansible.builtin.copy:
        src: "certs/vpnCert.pem"
        dest: "/etc/swanctl/x509/vpnCert.pem"
        mode: "644"

    - name: Copying VPN Key
      ansible.builtin.copy:
        src: "certs/vpnKey.pem"
        dest: "/etc/swanctl/private/vpnKey.pem"
        mode: "600"
  handlers:
    - name: Load Strongswan configuration
      ansible.builtin.shell: >
        swanctl --load-all
      changed_when: true
      listen: strongswan_config

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
        $mycert = ( Get-ChildItem -Path cert:LocalMachine\My | Where-Object { $_.Subject -Like "CN=win-srv.test.dk, O=TEST, C=DK" } )

        Add-VpnS2SInterface
        -Name "rtr.lego.dk"
        -Destination rtr.lego.dk
        -Protocol IKEv2
        -AuthenticationMethod MachineCertificates
        -ResponderAuthenticationMethod MachineCertificates
        -Certificate $mycert
        -Persistent
        -IPv4Subnet 10.10.0.0/24:10

        Set-VpnAuthProtocol
        -TunnelAuthProtocolsAdvertised Certificates
        -CertificateAdvertised $mycert

        Connect-VpnS2SInterface
        -Name "rtr.lego.dk"
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

<kbd>templates/strongswan-s2s-cert.j2</kbd>

```c
connections {
    s2s {
        local_addrs  = 20.0.0.2
        remote_addrs = win-srv.test.dk
        proposals = aes256-sha384-prfsha384-modp1024

        local {
            auth = pubkey
            certs = vpnCert.pem
        }
        remote {
            auth = pubkey
            id = "C=DK, O=TEST, CN=win-srv.test.dk"
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

include conf.d/*.conf
```