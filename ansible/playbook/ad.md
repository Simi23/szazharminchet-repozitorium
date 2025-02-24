---
title: Playbook: Active Directory setup
description: Active Directory setup with Ansible
published: true
date: 2025-02-24T11:15:16.280Z
tags: windows, ansible
editor: markdown
dateCreated: 2025-02-24T07:40:44.879Z
---

> For this guide you have to have your windows server assigned IPv4 address, and changed it's hostname.
{.is-info}

# Playbook

```yaml
---
- name: Create domain, promote to domain controller
  hosts: WIN-SRV-AD.TEST.DK
  gather_facts: false
  vars:
    ansible_user: Administrator
    ansible_become_user: Administrator
  tasks:
    - name: Create AD domain
      microsoft.ad.domain:
        safe_mode_password: Passw0rd
        dns_domain_name: TEST.DK
      register: domain
      changed_when: domain.reboot_required
      notify: Reboot

    - name: Run Reboot for Domain creation
      ansible.builtin.meta: flush_handlers
      
    - name: DC promote
      microsoft.ad.domain_controller:
        dns_domain_name: TEST.DK
        domain_admin_user: TEST\Administrator
        domain_admin_password: Passw0rd
        state: domain_controller
        safe_mode_password: Passw0rd
      register: dc
      changed_when: dc.reboot_required
      notify: Reboot

  handlers:
    - name: Reboot
      win_reboot:
      listen: Reboot

```

# Inventory

```yaml
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
