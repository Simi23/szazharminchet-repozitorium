---
title: Playbook: Backup Script
description: Backup script playbook cron scheduling
published: true
date: 2025-03-12T14:50:00.632Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-03-12T14:50:00.632Z
---

# Playbook

```yaml
- name: Configuring file backup
  hosts: backup
  tasks:
    - name: Creating script
      ansible.builtin.template:
        src: configs/backup.sh.j2
        dest: /opt/backup.sh
        mode: "755"
    
    - name: Creating cron job
      ansible.builtin.cron:
        name: backup_job
        minute: "0"
        job: "/bin/bash /opt/backup.sh"


```

# Inventory

```yml
backup:
  hosts:
    mail.dmz.worldskills.org:
  vars:
    backup:
      dirs:
        - configs
        - mailboxes
    directories:
      - src: /etc/postfix
        dest: configs/
      - src: /etc/dovecot
        dest: configs/
      - src: /mailboxes/*
        dest: mailboxes/
```

# Config files

```bash
#!/bin/bash
curdate=`date +"%Y-%m-%d---%H-%M-%S"`
{% for d in backup.dirs %}

mkdir -p "/opt/backup/$curdate/{{ d }}"
{% endfor %}
{% for d in directories %}

cp -R {{ d.src }} "/opt/backup/$curdate/{{ d.dest }}"
{% endfor %}

```