---
title: Playbook: Linux initial configuration
description: Linux initial SSH key copy 
published: true
date: 2025-02-24T08:44:38.551Z
tags: 
editor: markdown
dateCreated: 2025-02-24T08:42:11.465Z
---

# Playbook
```yaml
---
- name: Gathering and distributing SSH keys
  hosts: debian_servers
  become: true
  gather_facts: false
  tasks:
     - name: Scanning SSH keys
       delegate_to: localhost
       ansible.builtin.shell:
         cmd: ssh-keyscan -H {{ ansible_host }} >> ~/.ssh/known_hosts
       changed_when: false

     - name: Distributing SSH keys
       delegate_to: localhost
       ansible.builtin.command:
         cmd: sshpass -p "a" ssh-copy-id root@{{ ansible_host }} 
       changed_when: false

- name: Removing duplicate SSH keys
  hosts: debian_servers
  become: true
  gather_facts: false
  tasks:
    - name: Removing duplicate SSH keys
      delegate_to: localhost
      ansible.builtin.command:
        cmd: sort -u -o /root/.ssh/known_hosts /root/.ssh/known_hosts
      changed_when: false
```

# Inventory

```c
debian_severs:
	hosts:
  	LIN-RTR.lego.dk:
    	ansible_host: 10.10.0.254
      
    LIN-SRV1.lego.dk:
    	ansible_host: 10.10.0.101
    
    LIN-SRV2.lego.dk:
    	ansible_host: 10.10.0.102
```