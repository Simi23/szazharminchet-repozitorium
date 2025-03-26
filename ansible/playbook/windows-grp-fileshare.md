---
title: Playbook: Group creation, file sharing
description: Windows File Sharing in Ansible
published: true
date: 2025-03-26T11:32:22.905Z
tags: windows, ansible
editor: markdown
dateCreated: 2025-03-24T08:44:56.003Z
---

# Files
```
/opt/ansible
| - manage-shares.yaml
| - ansible.cfg
| - /inventory
    | - ansible_hosts
    | - group_vars
```

# Playbook
```yaml
---
- name: Manage Windows File Shares
  hosts: all
  become: true
  gather_facts: false
  vars_files:
    - inventory/group_vars
  tasks:
    # | Ensure AD groups exitsts (1/2) |
    - name: Ensure AD groups exitsts (1/2)
      microsoft.ad.group:
        name: "{{ item.write }}"
        scope: global
        state: present
      loop: "{{ file_shares }}"
    
    # | Ensure AD groups exitsts (2/2) |
    - name: Ensure AD groups exitsts (2/2)
      microsoft.ad.group:
        name: "{{ item.read }}"
        scope: global
        state: present
      loop: "{{ file_shares }}"
    
    # | Create folders |
    - name: Create sub folders
      ansible.windows.win_file:
        path: C:\project_share\{{ item.name }}
        state: directory
        type: allow
      loop: "{{ file_shares }}"

    # | Change Read NTFS permissions |
    - name: Change Read NTFS permissions
      ansible.windows.win_acl:
        path: C:\project_share\{{ item.name }}
        user: "{{ item.read }}"
        rights: ReadAndExecute
        type: allow
      when: item.read != item.write 
      loop: "{{ file_shares }}"

    # | Change Write NTFS permissions |
    - name: Change Write NTFS permissions
      ansible.windows.win_acl:
        path: C:\project_share\{{ item.name }}
        user: "{{ item.write }}"
        rights: Modify
        type: allow
      loop: "{{ file_shares }}"

    # | Create shares |
    - name: Create shares
      ansible.windows.win_share:
        name: "{{ item.name }}"
        path: C:\project_share\{{ item.name }}
        read: "{{ item.read }}"
        full: "{{ item.write }}"
      loop: "{{ file_shares }}"
   
```

# Inventory

### `ansible.cfg`
```yaml
[defaults]
inventory = inventory/ansible_hosts
host_key_checking = False
```

### `ansible_hosts`
```yaml
all:
  hosts:
    FILE-SRV:
      ansible_host: FILE-SRV.PARIS.LOCAL
      ansible_user: Administrator@paris.local
      ansible_password: Passw0rd
      ansible_become_user: PARIS\Administrator
      ansible_become_password: Passw0rd
    
    DC02.LYON.PARIS.LOCAL:
      ansible_host: 10.40.0.10
      ansible_user: Administrator@lyon.paris.local
      ansible_password: Passw0rd1
      ansible_become_user: LYON\Administrator
      ansible_become_password: Passw0rd1
      
  
  vars:
    ansible_port: 5985
    ansible_connection: winrm
    ansible_winrm_transport: ntlm
    ansible_become_method: runas
  
```

### `group_vars`
```yaml
file_shares:
  - name: "A"
    read: "g1"
    write: "g2"

  - name: "B"
    read: "g2"
    write: "g3"

  - name: "C"
    read: "g3"
    write: "g4"

```