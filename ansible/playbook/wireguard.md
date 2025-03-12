---
title: Playbook: Wireguard
description: Wireguard configuration for n client.
published: true
date: 2025-03-12T15:14:59.355Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-03-12T15:14:59.354Z
---

# Playbook

Main playbook:
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

wg-clt.yml:

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
      pub_add: 1.1.1.10

      wg_listen_port: 51820
      wg_srv_priv_path: /etc/wireguard/server.key
      wg_srv_pub_path: /etc/wireguard/server.pub 
      wg_psk_path: /etc/wireguard/psk
      srv_vpnv4: 10.1.30.1/24
      srv_vpnv6: 2001:db8:1001:30::1/64
      wg_peer:
        - name: clt1
          v4: 10.1.30.2/32
          v6: 2001:db8:1001:30::2/128
        - name: clt2
          v4: 10.1.30.2/32
          v6: 2001:db8:1001:30::2/128
      
      wg_clt_add:
        - name: clt1
          v4: 10.1.30.2/32
          v6: 2001:db8:1001:30::2/128
        - name: clt2
          v4: 10.1.30.3/32
          v6: 2001:db8:1001:30::3/128
```

# Config files

wg-clt.j2
```cfg
[Interface]
Address = {{ item.v4 }}
Address = {{ item.v6 }}
PrivateKey = {{ wg_clt_priv.stdout }}

[Peer]
PublicKey = {{ wg_srv_pub.stdout }}
PresharedKey = {{ wg_psk.stdout }}
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = {{ pub_add }}:{{ wg_listen_port }}
```

wg-srv.j2
```cfg
[Interface]
Address = {{ srv_vpnv4 }}
Address = {{ srv_vpnv6 }}
ListenPort = {{ wg_listen_port }}
PrivateKey = {{ wg_srv_priv.stdout }}

```