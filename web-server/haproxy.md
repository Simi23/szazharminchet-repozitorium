---
title: HAProxy
description: HAProxy configuratinon with high availibilty and FHRP
published: true
date: 2025-03-11T08:24:00.220Z
tags: linux
editor: markdown
dateCreated: 2025-03-10T09:43:05.489Z
---

# HAProxy

## Install
Install haproxy for your server. If you have another daemon that uses web ports you have to disable it!
```
apt install haproxy
```

## Configuration
Edit the configuration in `/etc/haproxy/haproxy.cfg`. Add these lines to the end:
```
...

frontend http-in
	bind *80,:::80 v6only
	redirect scheme https if !{ ssl_fc }
  
frontend https-in
	bind *443,:::443 v6only ssl crt /cert/web.pem
	option forwardfor
	http-response add-header via-proxy %[hostname]
	default_backend web_servers
  
backend web_servers
	balance roundrobin
	server	web01	web01.company.com:80 check
	server	web02	web02.company.com:80 check
```

> You have to include your full chain in your certificate file in the order:
> --> private key
> --> certificate
> --> sub certificate (if you have)
> --> root CA
>{.is-warning}

Restart the service
```
systemctl restart haproxy
```

## Keepalived (FHRP)
Install the service using apt.
```
apt install keepalived
```

### Primary server
Edit `/etc/keepalived/keepalived.conf`:
```
vrrp_script chk_haproxy {
	script "nc -zv localhost 80"
  interval 2
}

vrrp_instance VI_1 { 
	state MASTER 
	interface ens33 
	virtual_router_id 51 
	priority 100	
	advert_int 1	
	unicast_src_ip 10.1.20.21
  unicast_peer {
		10.1.20.22
	}
	authentication {
		auth_type PASS
		auth_pass Passw0rd
	}
	virtual_ipaddress {
		10.1.20.20
	}
	track_script {
		chk_haproxy
	}
}

vrrp_instance VI_2 {
	state MASTER 
	interface ens33 
	virtual_router_id 52 
	priority 100
	advert_int 1
	unicast_src_ip 2001:db8:1001:20::21
  unicast_peer {
		2001:db8:1001:20::22
	}
	authentication {
		auth_type PASS
		auth_pass Passw0rd
	}
	virtual_ipaddress {
		2001:db8:1001:20::20
	}
	track_script {
		chk_haproxy
	}
}
```

### Backup server
Edit `/etc/keepalived/keepalived.conf`:
```
vrrp_script chk_haproxy {
	script "nc -zv localhost 80"
  interval 2
}

vrrp_instance VI_1 { 
	state BACKUP 
	interface ens33
	virtual_router_id 51 
	priority 90
	advert_int 1
	unicast_src_ip 10.1.20.22
  unicast_peer {
		10.1.20.21
	}
	authentication {
		auth_type PASS
		auth_pass Passw0rd
	}
	virtual_ipaddress {
		10.1.20.20
	}
	track_script {
		chk_haproxy
	}
}

vrrp_instance VI_2 {
	state BACKUP 
	interface ens33 
	virtual_router_id 52 
	priority 90
	advert_int 1
	unicast_src_ip 2001:db8:1001:20::22
  unicast_peer {
		2001:db8:1001:20::21
	}
	authentication {
		auth_type PASS
		auth_pass Passw0rd
	}
	virtual_ipaddress {
		2001:db8:1001:20::20
	}
	track_script {
		chk_haproxy
	}
}
```

Restart the service
```
systemctl restart keepalived
```