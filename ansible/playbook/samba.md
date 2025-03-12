---
title: Playbook: Samba
description: Samba playbook with user creation
published: true
date: 2025-03-12T14:47:21.323Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-03-12T14:47:21.323Z
---

# Playbook

```yaml
- name: Configuring Samba server
  hosts: samba
  tasks:
    - name: Installing Samba
      ansible.builtin.apt:
        name: samba
        state: present
    
    - name: Creating base directory
      ansible.builtin.file:
        path: /srv/samba
        state: directory
        mode: "0777"
    
    - name: Creating share directories
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: directory
        mode: "0777"
      loop: "{{ shares }}"
      loop_control:
        label: "{{ item.path }}"
    
    - name: Creating samba config
      ansible.builtin.template:
        src: configs/smb.conf.j2
        dest: /etc/samba/smb.conf
        mode: "0644"
      notify: Restarting samba

    - name: Creating users
      ansible.builtin.user:
        name: "{{ item.name }}"
        group: users
        password: "{{ item.password | password_hash('sha512') }}"
        update_password: on_create
      loop: "{{ users }}"

    - name: Checking for samba users
      ansible.builtin.shell: pdbedit -L | grep -e "^{{ item.name }}:"
      loop: "{{ users }}"
      register: smb_check
      changed_when: smb_check.rc != 0
      failed_when: false
      notify: Creating samba users

  handlers:
    - name: Creating samba users
      ansible.builtin.shell: printf "{{ item.password }}\n{{ item.password }}" | smbpasswd -sa {{ item.name }}
      loop: "{{ users }}"

    - name: Restarting samba
      ansible.builtin.service:
        name: smbd
        state: restarted

```

# Inventory

```yml
samba:
  hosts:
    int-srv01.int.worldskills.org:
  vars:
    users:
      - name: jamie
        password: Passw0rd
      - name: helloworld
        password: Passw0rd

    shares:
      - name: public
        path: /srv/samba/public
        read_only: no
        write_list: jamie helloworld
        guest_ok: yes

      - name: internal
        path: /srv/samba/internal
        read_only: no
        write_list: jamie helloworld
        read_list: jamie helloworld
        create_mode: "0777"
        dir_mode: "0777"
```

# Config files

```ini
[global]
   workgroup = WORKGROUP
   log file = /var/log/samba/log.%m
   max log size = 1000
   logging = file
   panic action = /usr/share/samba/panic-action %d
   server role = standalone server
   obey pam restrictions = yes
   unix password sync = yes
   passwd program = /usr/bin/passwd %u
   passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .
   pam password change = yes
   map to guest = bad user
   usershare allow guests = yes

[homes]
   comment = Home Directories
   browseable = no
   read only = yes
   create mask = 0700
   directory mask = 0700
   valid users = %S

[printers]
   comment = All Printers
   browseable = no
   path = /var/tmp
   printable = yes
   guest ok = no
   read only = yes
   create mask = 0700

[print$]
   comment = Printer Drivers
   path = /var/lib/samba/printers
   browseable = yes
   read only = yes
   guest ok = no

{% for share in shares %}

[{{ share.name }}]
   path = {{ share.path }}
   read only = {{ share.read_only }}

   {% if share.read_list is defined %}read list = {{ share.read_list }}{% endif %}

   {% if share.write_list is defined %}write list = {{ share.write_list }}{% endif %}

   {% if share.guest_ok is defined %}guest ok = {{ share.guest_ok }}{% endif %}

   {% if share.create_mode is defined %}force create mode = {{ share.create_mode }}{% endif %}

   {% if share.dir_mode is defined %}force directory mode = {{ share.dir_mode }}{% endif %}

{% endfor %}

```