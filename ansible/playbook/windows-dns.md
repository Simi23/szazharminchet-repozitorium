---
title: Playbook: Windows DNS setup
description: Windows DNS server setup using Ansible
published: true
date: 2025-03-01T14:04:14.037Z
tags: windows, ansible
editor: markdown
dateCreated: 2025-02-24T07:53:15.115Z
---

# Playbook

```yaml
---
- name: Check default zone, create Reverse zone, set forwarder
  hosts: WIN-SRV-AD.TEST.DK
  gather_facts: false
  tasks:
    - name: Check if DNS Forward lookup zone exists
      win_shell: |
        if (Get-DnsServerZone -Name test.dk -ErrorAction SilentlyContinue) {
          Write-Output "ZoneExists"
        } else {
          Write-Output "ZoneDoesNotExist"
        }
      register: dns_zone_check
      changed_when: '"ZoneExists" not in dns_zone_check.stdout'
      notify: Create Forward

    - name: Check if DNS Reverse lookup zone exists
      win_shell: |
        if (Get-DnsServerZone -Name 0.20.10.in-addr.arpa -ErrorAction SilentlyContinue) {
          Write-Output "ZoneExists"
        } else {
          Write-Output "ZoneDoesNotExist"
        }
      register: dns_zone_check
      changed_when: '"ZoneExists" not in dns_zone_check.stdout'
      notify: Create Reverse

    - name: Create Zones
      ansible.builtin.meta: flush_handlers

    - name: Check if DNS A record exists
      win_shell: "nslookup 10.20.0.254"
      register: nslookup_check
      changed_when: '"win-srv.test.dk" not in nslookup_check.stdout'
      notify: Create A

    - name: Check if DNS forwarder exists
      win_shell: >
        $forwarder = Get-DnsServerForwarder

        if ($forwarder) {
          Write-Output "ForwarderExists"
        } else {
          Write-Output "ForwarderDoesNotExist"
        }
      register: dns_forwarder_check
      changed_when: '"ForwarderDoesNotExist" in dns_forwarder_check.stdout'
      notify: Add forwarder

  handlers:
    - name: Create Forward lookup zones
      win_shell: >
        Add-DnsServerPrimaryZone -Name "{{ item.name }}" -ReplicationScope "Forest"
      loop: "{{ win_dns.forwardzones }}"
      listen: Create Forward

    - name: Create Forward lookup zones
      win_shell: >
        Add-DnsServerPrimaryZone -Name "{{ item.name }}" -ReplicationScope "Forest"
      loop: "{{ win_dns.reversezones }}"
      listen: Create Reverse

    - name: Create A records
      win_shell: >
        Add-DnsServerResourceRecordA -ZoneName "{{item.zonename}}" -Name "{{ item.name }}" -IPv4Address "{{ item.value }}" -CreatePtr
      loop: "{{ win_dns.a_records }}"
      listen: Create A

    - name: Add DNS forwarder
      win_shell: >
        Add-DnsServerForwarder -IPAddress 20.0.0.1 -PassThru
      listen: Add forwarder
```

# Inventory

```yaml
all:
	vars:
		win_dns:
      forwardzones:
        - name: test.dk

      reversezones:
        - name: 0.20.10.in-addr.arpa

      a_records:
        - name: win-srv
          value: "10.20.0.254"
          zonename: "test.dk"

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