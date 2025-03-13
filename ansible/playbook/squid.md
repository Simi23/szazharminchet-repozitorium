---
title: Playbook: Squid
description: Transparent proxy configuration with HTTP
published: true
date: 2025-03-13T07:58:38.492Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-03-12T15:17:14.504Z
---

# Playbook
```yaml
---
- name: Install and configure squid
  hosts: fw
  gather_facts: false
  tasks:
    # | Install Squid |
    - name: Install Squid
      ansible.builtin.apt:
        name: squid
        state: present

    # | Copy configuration file |
    - name: Copy configuration file
      ansible.builtin.template:
        src: configs/squid.j2
        dest: /etc/squid/conf.d/squid-transparent.conf
      notify: Restart

  handlers:
    # | Reload Squid |
    - name: Reload Squid
      ansible.builtin.service:
        name: squid
        state: reloaded
      listen: Restart

```

# Inventory

```yml
all:
  vars:
    ansible_user: root
    ansible_password: a

firewall:
  hosts:
    fw:
      ansible_host: 10.1.20.1
      squid_header: x-secured-by "clearsky-proxy"
      
```

# Config files

```conf
http_access allow all
http_port 3128 intercept
http_port 3129

reply_header_add {{ squid_header }}

shutdown_lifetime 2 seconds

```