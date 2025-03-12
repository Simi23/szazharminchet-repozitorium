---
title: Playbook: SSH Certificates
description: SSH certificate playbook
published: true
date: 2025-03-12T14:53:13.166Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-03-12T14:53:13.166Z
---

# Playbook

```yaml
- name: Creating SSH CA and user certificates
  hosts: localhost
  tasks:
    - name: Creating SSH CA key
      ansible.builtin.command:
        cmd: ssh-keygen -t rsa -N "" -b 4096 -f {{ base }}/ca
        creates: "{{ base }}/ca"
    
    - name: Creating client keys
      ansible.builtin.command:
        cmd: ssh-keygen -t rsa -N "" -b 4096 -f {{ base }}/{{ item.principal }}
        creates: "{{ base }}/{{ item.principal }}"
      loop: "{{ users }}"
    
    - name: Signing client keys
      ansible.builtin.command:
        cmd: >
          ssh-keygen
          -s {{ base }}/ca
          -I {{ item.identity }}
          -n {{ item.principal }}
          -V +52w
          {{ base }}/{{ item.principal }}.pub
        creates: "{{ base }}/{{ item.principal }}-cert.pub"
      loop: "{{ users }}"

- name: Configuring servers to accept the CA
  hosts: ssh_servers
  tasks:
    - name: Copying the CA certificate
      ansible.builtin.copy:
        src: ssh/ca.pub
        dest: /etc/ssh/ca.pub
        mode: "0644"
      notify: Restarting SSHD
    
    - name: Creating configuration
      ansible.builtin.copy:
        src: configs/sshd_ca
        dest: /etc/ssh/sshd_config.d/sshd_ca.conf
        mode: "0644"
      notify: Restarting SSHD
  
  handlers:
    - name: Restarting SSHD
      ansible.builtin.service:
        name: sshd
        state: restarted

- name: Configuring clients to use the CA
  hosts: ssh_clients
  tasks:
    - name: Copying private keys
      ansible.builtin.copy:
        src: "{{base}}/{{item.principal}}"
        dest: "{{item.ssh_dir}}/"
        mode: "0600"
      loop: "{{ copy }}"

    - name: Copying certificates
      ansible.builtin.copy:
        src: "{{base}}/{{item.principal}}-cert.pub"
        dest: "{{item.ssh_dir}}/"
        mode: "0600"
      loop: "{{ copy }}"

    - name: Creating configs
      ansible.builtin.template:
        src: "configs/ssh_config.j2"
        dest: "{{config.location}}/config"
        mode: "0644"

```

# Inventory

```yml
ssh:
  children:
    ssh_servers:
    ssh_clients:
  hosts:
    localhost:
  vars:
    base: /home/administrator/ansible/ssh
    users:
      - identity: root@dmz.worldskills.org
        principal: root

ssh_servers:
  hosts:
    mail.dmz.worldskills.org:

ssh_clients:
  hosts:
    ha-prx01.dmz.worldskills.org:
      copy:
        - principal: root
          ssh_dir: /root/.ssh
      config:
        location: /root/.ssh
        hosts:
          - host: mail.dmz.worldskills.org
            username: root
            filename: root
            ssh_dir: /root/.ssh

```

# Config files

<kbd>configs/sshd_ca</kbd>

```bash
TrustedUserCAKeys /etc/ssh/ca.pub
```

<kbd>configs/ssh_config.j2</kbd>

```ini
{% for h in config.hosts %}

Host {{ h.host }}
    HostName {{ h.host }}
    User {{ h.username }}
    IdentityFile {{ h.ssh_dir }}/{{ h.filename }}
    CertificateFile {{ h.ssh_dir }}/{{ h.filename }}-cert.pub

{% endfor %}
```