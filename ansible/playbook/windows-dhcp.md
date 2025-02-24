---
title: Playbook: Windows DHCP server setup
description: Windwos DHCP server setup with Ansible
published: true
date: 2025-02-24T07:47:32.305Z
tags: windows, ansible
editor: markdown
dateCreated: 2025-02-24T07:47:32.305Z
---

# Playbook

```yaml
---
- name: Windows DHCP server setup
  hosts: WIN-SRV-AD.TEST.DK
  gather_facts: false
  tasks:
    - name: Install DHCP feature and management tools
      ansible.windows.win_feature:
        name: 
          - DHCP
          - RSAT-DHCP
        state: present
        include_management_tools: true
      register: dhcp_feature_result
      changed_when: dhcp_feature_result.reboot_required
      notify: Reboot

    - name: Reboot if needed
      ansible.builtin.meta: flush_handlers

    - name: Authorize DHCP server to domain
      ansible.windows.win_shell: |
        if (!(Get-DhcpServerInDC | Where-Object { $_.DnsName -eq "WIN-SRV-AD.TEST.DK" })) {
          Write-Output "DHCPnotauthorized"
        } else {
          Write-Output "DHCPauthorized"
        }
      register: dhcp_authorize
      changed_when: '"DHCPauthorized" not in dhcp_authorize.stdout'
      notify: Authorize

    - name: Authorize if needed
      ansible.builtin.meta: flush_handlers

    - name: Register clients to DNS server
      ansible.windows.win_shell: |
        $credential = New-Object System.Management.Automation.PSCredential("TEST\Administrator", (ConvertTo-SecureString "Passw0rd" -AsPlainText -Force))

        $existingCredential = Get-DhcpServerDnsCredential -ErrorAction SilentlyContinue
        if (!$existingCredential) {
          Set-DhcpServerDnsCredential -Credential $credential -ComputerName "WIN-SRV-AD.TEST.DK"
        }
      changed_when: false

    - name: Configure IPv4 scopes
      win_shell: |
        $scope = Get-DhcpServerv4Scope -ScopeId "{{ item.scopeid }}" -ErrorAction SilentlyContinue

        if (-not $scope) {
          Add-DhcpServerv4Scope -Name "{{ item.name }}" -StartRange "{{ item.start }}" -EndRange "{{ item.end }}" -SubnetMask "{{ item.netmask }}" -State Active
          Set-DhcpServerv4OptionValue `
          -ScopeId "{{ item.scopeid }}" `
          -ComputerName "WIN-SRV-AD.TEST.DK" `
          -DnsServer "{{ item.dns }}" `
          -WinsServer "{{ item.wins }}" `
          -DnsDomain "TEST.DK" `
          -Router "{{ item.gateway }}"
        } else {
          Write-Host "Scope {{ item.scopeid }} already exists, skipping creation."
        }
      loop: "{{ dhcp.scopev4 }}"
      loop_control:
        label: "{{ item.name }}"
      register: scope_create
      changed_when: '"already exists, skipping creation." not in scope_create.stdout'

  handlers:
    - name: Reboot the server
      ansible.windows.win_reboot:
      listen: Reboot

    - name: Authorize DHCP server
      ansible.windows.win_shell: |
        Add-DhcpServerInDC -DnsName "WIN-SRV-AD.TEST.DK"  -IPAddress "{{ ipv4_add }}"
      listen: Authorize
    
    - name: Notify server manager that Post-install is done
      ansible.windows.win_shell: |
        $currentState = (Get-ItemProperty -Path registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\ServerManager\Roles\12 -Name ConfigurationState).ConfigurationState
        
        if ($currentState -ne 2) {
          Set-ItemProperty -Path registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\ServerManager\Roles\12 -Name ConfigurationState -Value 2
        }
      listen: Authorize
```

# Inventory

```yaml
all:
  vars:
    dhcp:
      scopev4:
        - name: "WIN-CLIENT"
          start: "10.20.0.10" 
          end: "10.20.0.240"
          netmask: "255.255.255.0"
          scopeid: "10.20.0.0"
          dns: "10.20.0.254"
          wins: "10.20.0.254"
          gateway: "10.20.0.254"

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
