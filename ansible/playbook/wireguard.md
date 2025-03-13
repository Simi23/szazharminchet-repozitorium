---
title: Playbook: Wireguard
description: Wireguard configuration for n client.
published: true
date: 2025-03-13T08:23:43.862Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-03-12T15:14:59.354Z
---

# Playbook

Main playbook:
```yaml
---
- name: Create wireguard config files
  hosts: fw
  gather_facts: false
  tasks:
    # | Install Wireguard |
    - name: Install Wireguard
      ansible.builtin.apt:
        name:
          - wireguard
          - wireguard-tools
        state: present
      notify: Install

    
  handlers:
    # | Generate private key |
    - name: Generate private key
      ansible.builtin.shell: wg genkey > /etc/wireguard/server.key
      listen: Install

    # | Generate public key |
    - name: Generate public key
      ansible.builtin.shell: wg pubkey < /etc/wireguard/server.key > /etc/wireguard/server.pub
      listen: Install

    # | Generate psk |
    - name: Generate psk
      ansible.builtin.shell: wg genpsk > /etc/wireguard/psk
      listen: Install

    # | Get private key |
    - name: Get private key
      ansible.builtin.command: cat /etc/wireguard/server.key
      register: wg_srv_priv
      listen: Install

    # | Get public key |
    - name: Get public key
      ansible.builtin.command: cat /etc/wireguard/server.pub
      register: wg_srv_pub
      listen: Install
    
    # | Get psk |
    - name: Get psk
      ansible.builtin.command: cat /etc/wireguard/psk
      register: wg_psk
      listen: Install 

    # | Copy main config file |
    - name: Copy main config file
      ansible.builtin.template:
        src: configs/wg-srv.j2
        dest: /etc/wireguard/wg0.conf
      listen: Install


    # | Add client configurations |
    - name: Add client configurations
      include_tasks: wg-clt.yml
      loop: "{{ wg_clt_add }}"
      loop_control:
        loop_var: item
      listen: Install


    # | Start the service |
    - name: Start the service
      ansible.builtin.command: wg-quick up wg0
      listen: Install

    # | Enable the service |
    - name: Enable the service
      ansible.builtin.service:
        name: wg-quick@wg0
        enabled: true
      listen: Install
      
```

wg-clt.yml:
```
# | Create directory for client config|
- name: Create peers configuration
  ansible.builtin.file:
    path: /etc/wireguard/{{ item.name }}
    state: directory
    mode: '0777'

# | Create private key |
- name: Create private key
  ansible.builtin.shell: wg genkey > /etc/wireguard/{{ item.name }}/client.key

# | Create public key |
- name: Create public key
  ansible.builtin.shell: wg pubkey < /etc/wireguard/{{ item.name }}/client.key > /etc/wireguard/{{ item.name }}/client.pub

# | Get private key |
- name: Get private key
  ansible.builtin.command: cat /etc/wireguard/{{ item.name }}/client.key
  register: wg_clt_priv

# | Get public key |
- name: Get public key
  ansible.builtin.command: cat /etc/wireguard/{{ item.name }}/client.pub
  register: wg_clt_pub

# | Add peer information to servers config |
- name: Add peer information to servers config
  ansible.builtin.shell: printf "\n[Peer]\nAllowedIPs={{ item.v4 }},{{ item.v6 }}\nPresharedKey={{ wg_psk.stdout }}\nPublicKey={{ wg_clt_pub.stdout }}\n" >> /etc/wireguard/wg0.conf

# | Create peers configuration |
- name: Create peers configuration
  ansible.builtin.template:
    src: configs/wg-clt.j2
    dest: /etc/wireguard/{{ item.name }}/wg0.conf

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