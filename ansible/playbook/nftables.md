---
title: Playbook: Nftables
description: Nftables rule setup with ansible
published: true
date: 2025-03-12T15:18:43.343Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-03-12T15:18:43.343Z
---

# Playbook

Main playbook:
```yaml
---
- name: Install and configure nftables
  hosts: fw
  gather_facts: false
  tasks:
    # | Install Nftables |
    - name: Install Nftables
      ansible.builtin.apt:
        name: nftables
        state: present

    # | Copy configuration file |
    - name: Copy configuration file
      ansible.builtin.template:
        src: configs/nft.j2
        dest: /etc/nftables.conf
      notify: Restart

  handlers:
    # | Enable, Reload Nftables |
    - name: Enable, Reload Nftables
      ansible.builtin.service:
        name: nftables
        state: reloaded
        enabled: true
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
     
      nft_forward:
        - "policy drop;"
        - "ct state { established, related } accept;"
        - "ip saddr { 10.1.10.0/24, 10.1.20.0/24 } oif ens38 accept;"
        - "ip6 saddr { 2001:db8:1001:10::/64, 2001:db8:1001:20::/64} oif ens38 accept;"
        - "ip saddr 10.1.10.0/24 ip daddr 10.1.20.0/24 accept;"
        - "ip6 saddr 2001:db8:1001:10::/64 ip6 daddr 2001:db8:1001:20::/64 accept;"
        - "ip saddr 10.1.30.0/24 ip daddr { 10.1.10.0/24, 10.1.20.0/24 } accept;"
        - "ip6 saddr 2001:db8:1001:30::/64 ip6 daddr { 2001:db8:1001:10::/64, 2001:db8:1001:20::/64 } accept;"
        - "ip saddr 10.1.20.10 ip daddr 10.1.10.10 tcp dport 389 accept;"
        - "ip6 saddr 2001:db8:1001:20::10 ip6 daddr 2001:db8:1001:10::10 tcp dport 389 accept;"

      nft_pre:
        - "ip daddr 1.1.1.10 tcp dport {80,443,53} dnat to 10.1.20.20;"
        - "ip daddr 1.1.1.10 udp dport 53 dnat to 10.1.20.20;"
        - "ip saddr { 10.1.10.0/24, 10.1.30.0/24 } tcp dport 80 redirect to 3128;"
        - "ip6 saddr { 2001:db8:1001:10::/64, 2001:db8:1001:30::/64 } tcp dport 80 redirect to 3128;"

      nft_post:
        - "ip saddr { 10.1.10.0/24, 10.1.20.0/24 } oif ens38 masquerade;"
```

# Config files

```conf
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain forward {
        type filter hook forward priority filter;
        {% for item in nft_forward %}
        {{ item }}
        {% endfor %}
    }

    chain dstnat {
        type nat hook prerouting priority dstnat;
        {% for item in nft_pre %}
        {{ item }}
        {% endfor %}
    }

    chain srcnat {
        type nat hook postrouting priority srcnat;
        {% for item in nft_post %}
        {{ item }}
        {% endfor %}
    }
}

```