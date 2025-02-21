---
title: Playbook: Bind9 DNS Master/Slave
description: Bind9 DNS Ansible playbook with Master/Slave replication
published: true
date: 2025-02-21T06:53:27.751Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-02-21T06:53:27.751Z
---

# Playbook

```yaml
---
- name: Installing Bind9
  hosts: lin_servers
  become: true
  tasks:
    - name: Setting public DNS for package installation
      ansible.builtin.template:
        src: templates/resolv.conf.j2
        dest: /etc/resolv.conf
        mode: "644"
        lstrip_blocks: true
      vars:
        dns:
          - "20.0.0.1"
        domain: "localdomain"

    - name: Installing Bind9
      ansible.builtin.apt:
        name:
          - bind9
          - bind9-dnsutils
        state: present

    - name: Configuring DNS options
      ansible.builtin.template:
        src: templates/named.conf.options.j2
        dest: /etc/bind/named.conf.options
        mode: "644"
        lstrip_blocks: true
      notify: Restart Bind9

    - name: Configuring DNS zone options
      ansible.builtin.template:
        src: templates/named.conf.local.j2
        dest: /etc/bind/named.conf.local
        mode: "644"
        lstrip_blocks: true
      notify: Restart Bind9

    - name: Reading DNS Roles
      ansible.builtin.group_by:
        key: dns_server_{{ dns_role }}
      changed_when: false

  handlers:
    - name: Restart Bind9
      ansible.builtin.service:
        name: named
        state: restarted

- name: Configuring Master Server
  hosts: dns_server_master
  become: true
  tasks:
    - name: Configuring DNS Zones
      ansible.builtin.template:
        src: templates/named.zone.j2
        dest: "{{ item.file }}"
        mode: "644"
        lstrip_blocks: true
      loop: "{{ dns.zones }}"
      loop_control:
        label: "ZONE {{ item.name }}"
      vars:
        zone: "{{ item }}"
      notify: Restart Bind9

  handlers:
    - name: Restart Bind9
      ansible.builtin.service:
        name: named
        state: restarted

- name: Resetting resolver config
  hosts: lin_servers
  become: true
  tasks:
    - name: Setting local DNS
      ansible.builtin.template:
        src: templates/resolv.conf.j2
        dest: /etc/resolv.conf
        mode: "644"
        lstrip_blocks: true
      vars:
        dns:
          - "10.10.0.101"
          - "10.10.0.102"
        domain: "lego.dk"

    - name: Setting up hosts file
      ansible.builtin.template:
        src: templates/hosts.j2
        dest: /etc/hosts
        mode: "644"
        lstrip_blocks: true

```

# Inventory

```yaml
lin_servers:
  vars:
    ansible_user: root
    ansible_password: Passw0rd
    ansible_python_interpreter: auto_silent
    domain: lego.dk
  hosts:
    LIN-SRV1.LEGO.DK:
      ansible_host: 10.10.0.101
      address: 10.10.0.101
      hostname: lin-srv1
      dns_role: master
    LIN-SRV2.LEGO.DK:
      ansible_host: 10.10.0.102
      address: 10.10.0.102
      hostname: lin-srv2
      dns_role: slave

all:
  vars:
    dns:
      allow_forward: true
      forwarders:
        - 20.0.0.1
      masters:
        - 10.10.0.101
      slaves:
        - 10.10.0.102
      zones:
        - name: "lego.dk"
          file: "/etc/bind/db.lego.dk"
          records:
            - "@          NS  lin-srv1.lego.dk."
            - "@          NS  lin-srv2.lego.dk."
            - "lin-srv1   A   10.10.0.101"
            - "lin-srv2   A   10.10.0.102"
            - "lin-clt    A   10.10.0.1"
            - "rtr        A   10.10.0.254"

        - name: "0.10.10.in-addr.arpa"
          file: "/etc/bind/db.10.10.0"
          records:
            - "@    NS  lin-srv1.lego.dk."
            - "@    NS  lin-srv2.lego.dk."
            - "101  PTR lin-srv1.lego.dk."
            - "102  PTR lin-srv2.lego.dk."
            - "1    PTR lin-clt.lego.dk."
            - "254  PTR rtr.lego.dk."
```

# Templates

<kbd>templates/hosts.j2</kbd>

```c
127.0.0.1       localhost
{{ address }}   {{ hostname }}.{{ domain }} {{ hostname }}

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

<kbd>templates/named.conf.local.j2</kbd>

```c
{% for zone in dns.zones %}
zone "{{ zone.name }}" {
    type {{ hostvars[inventory_hostname]["dns_role"] }};
    {% if hostvars[inventory_hostname]["dns_role"] == "master" %}
        file "{{ zone.file }}";
        allow-transfer {
            {% for slave in dns.slaves %} 
                {{ slave }};
            {% endfor %}
        };
        also-notify { 
            {% for slave in dns.slaves %} 
                {{ slave }};
            {% endfor %}
        };
    {% else %}
        masters {
            {% for master in dns.masters %} 
                {{ master }};
            {% endfor %}
        };
    {% endif %}
};
{% endfor %}
```

<kbd>templates/named.conf.options.j2</kbd>

```c
options {
    directory "/var/cache/bind";

    {% if dns.allow_forward %}
        forwarders {
            {% for forwarder in dns.forwarders %}
                {{ forwarder }};
            {% endfor %}
        };
    {% endif %}

    dnssec-validation auto;

    listen-on-v6 { any; };
};
```

<kbd>templates/named.zone.j2</kbd>

```c
$TTL    604800
@       IN      SOA     {{ zone.name }}. root.{{ zone.name }}. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

{% for record in zone.records %}
{{ record }}
{% endfor %}
```

<kbd>templates/resolv.conf.j2</kbd>

```c
domain {{ domain }}
search {{ domain }}
{% for address in dns %}
nameserver {{ address }}
{% endfor %}
```
