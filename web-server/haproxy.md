---
title: HAProxy
description: HAProxy configuratinon with high availibilty and FHRP
published: true
date: 2025-03-10T09:58:14.895Z
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
.
.
.
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
  server	web01 web01.company.com:80 check
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

## Keepalived
Install the service using apt.
```
apt install keepalived
```

### Primary server
Edit `/etc/keepalived/keepalived.conf`:
```

```

### Backup server
Edit `/etc/keepalived/keepalived.conf`:
```

```