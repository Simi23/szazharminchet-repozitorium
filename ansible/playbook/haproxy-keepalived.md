---
title: Playbook: Reverse proxy
description: HaProxy, keepalived high availibilty configuartion with ansible
published: true
date: 2025-03-13T07:58:28.915Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-03-12T15:20:30.469Z
---

# Playbook

```yaml
---
- name: Install and configure Haproxy
  hosts: proxys
  gather_facts: false
  tasks:
    # | Install Haproxy |
    - name: Install Haproxy
      ansible.builtin.apt:
        name: haproxy
        state: present

    # | Copy configuration file to the server |
    - name: Copy configuration file to the server
      ansible.builtin.template:
        src: configs/haproxy.j2
        dest: /etc/haproxy/haproxy.conf
      notify: 
        - Restart
        - Append to file

  handlers:
    # | Copy configuration file into the config file |
    - name: Copy configuration file into the config file
      ansible.builtin.shell: cat /etc/haproxy/haproxy.conf >> /etc/haproxy/haproxy.cfg
      listen: Append to file

    # | Enable, Reload Haproxy |
    - name: Enable, Reload Haproxy
      ansible.builtin.service:
        name: haproxy
        state: restarted
        enabled: true
      listen: Restart


- name: Install and configure Keepalived
  hosts: proxys
  gather_facts: false
  tasks:
    # | Install keepalived |
    - name: Install keepalived
      ansible.builtin.apt:
        name: keepalived
        state: present

    # | Copy configuration file|
    - name: Copy configuration file
      ansible.builtin.template:
        src: configs/keepalived.j2
        dest: /etc/keepalived/keepalived.conf
      notify: Restart

  handlers:
    # | Enable, Reload keepalived |
    - name: Enable, Reload keepalived
      ansible.builtin.service:
        name: keepalived
        state: restarted
        enabled: true
      listen: Restart

```

# Inventory

```yml
all:
  vars:
    ansible_user: root
    ansible_password: a

proxys:
  hosts:
    ha-prx01:
      ansible_host: ha-prx01.dmz.worldskills.org
      priority: 100
      type: MASTER
      ipv4: 10.1.20.21
      ipv6: 2001:db8:1001:20::21
      peerv4: 10.1.20.22
      peerv6: 2001:db8:1001:20::22
    
    ha-prx02:
      ansible_host: ha-prx02.dmz.worldskills.org
      priority: 90
      type: BACKUP
      ipv4: 10.1.20.22
      ipv6: 2001:db8:1001:20::22
      peerv4: 10.1.20.21
      peerv6: 2001:db8:1001:20::21
  
  vars:
    virtual_addv4: 10.1.20.20
    virtual_addv6: 2001:db8:1001:20::20
    web_servers:
      - web01 web01.dmz.worldskills.org:80
      - web02 web02.dmz.worldskills.org:80
    pem_path: /cert/web.pem
    proxy_header: "via-proxy %[hostname]"
    interface: ens33
    password: Passw0rd
```

# Config files

haproxy.j2:
```conf


frontend http-in
    bind :::80
    redirect scheme https if !{ ssl_fc }

frontend https-in
    bind :::443 ssl crt {{ pem_path }}
    option forwardfor
    http-response add-header {{ proxy_header }}
    default_backend web_servers

backend web_servers
    balance roundrobin
    {% for item in web_servers %}
    server  {{ item }} check
    {% endfor %}
    
```

keepalived.j2:
```conf
vrrp_script chk_haproxy {
    script "nc -zv localhost 80"
    interval 2
}

vrrp_instance v4 {
    type {{ type }}
    interface {{ interface }}
    virtual_router_id 51
    priority {{ priority }}
    advert_int 1
    unicast_src_ip {{ ipv4 }}
    unicast_peer {
        {{ peerv4 }}
    }
    authentication {
        auth_type PASS
        auth_pass {{ password }}
    }
    virtual_ipaddress {
        {{ virtual_addv4 }}
    }
    track_script {
        chk_haproxy
    }
}

vrrp_instance v6 {
    type {{ type }}
    interface {{ interface }}
    virtual_router_id 52
    priority {{ priority }}
    advert_int 1
    unicast_src_ip {{ ipv6 }}
    unicast_peer {
        {{ peerv6 }}
    }
    authentication {
        auth_type PASS
        auth_pass {{ password }}
    }
    virtual_ipaddress {
        {{ virtual_addv6 }}
    }
    track_script {
        chk_haproxy
    }
}

```