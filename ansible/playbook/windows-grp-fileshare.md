---
title: Playbook: Group creation, file sharing
description: Windows File Sharing in Ansible
published: true
date: 2025-03-24T08:56:17.514Z
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
```
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
      ansible.windows.win_group:
        name: "{{ item.read }}"
        state: present
        getent: yes
      loop: "{{ file_shares }}"
    
    # | Ensure AD groups exitsts (2/2) |
    - name: Ensure AD groups exitsts (2/2)
      ansible.windows.win_group:
        name: "{{ item.write }}"
        state: present
        getent: yes
      loop: "{{ file_shares }}"

    # | Create the directory for project shares |
    - name: Create the directory for project shares
      ansible.windows.win_file:
        path: "C:\\project_share\\{{ item.name }}"
        state: directory
      loop: "{{ file_shares }}"

    # | Create Windows file shares |
    - name: Create Windows file shares
      ansible.windows.win_share:
        name: "{{ item.name }}"
        path: "C:\\project_share\\{{ item.name }}"
        full: Everyone
        state: present
      loop: "{{ file_shares }}"

    # | Set NTFS read permissions |
    - name: Set NTFS read permissions
      ansible.windows.win_acl:
        path: "C:\\project_share\\{{ item.name }}"
        user: "{{ item.read }}"
        rights: ReadAndExecute
        type: allow
        state: present
      loop: "{{ file_shares }}"

    # | Set NTFS write permissions |
    - name: Set NTFS write permissions
      ansible.windows.win_acl:
        path: "C:\\project_share\\{{ item.name }}"
        user: "{{ item.write }}"
        rights: Modify
        type: allow
        state: present
      loop: "{{ file_shares }}"

```

# Inventory