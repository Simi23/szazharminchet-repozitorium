---
title: Squid Transparent Proxy
description: Squid Transparent Proxy configuration
published: true
date: 2025-03-10T09:57:25.661Z
tags: linux
editor: markdown
dateCreated: 2025-03-10T09:57:25.661Z
---

# Installation

Install the package.

```bash
apt install squid
```

# Configuration

The existing configuration file contains the whole documentation. You should create a backup of it, then edit a new file.

For a simple HTTP transparent proxy, you need the following configuration.

```ini
http_access allow all
http_port 3128 intercept
http_port 3129
```

> For the transparent mechanism, you need the `http_port` line with the `intercept` option. However, you still need the regular proxy service to listen as well, otherwise the service will not start.
{.is-warning}

You can also add headers to the HTTP responses:

```ini
reply_header_add x-secured-by "clearsky-proxy"
```

## Firewall configuration

The transparent proxy is called transparent because the client doesn't know about it, which means you have to route the packets towards the proxy, which were originally destined for the web.

This NFTables example shows a scenario where the router is also the proxy.

```c
...

table inet filter {
	...
  
  chain portfw {
  	type nat hook prerouting priority dstnat;
    ip saddr { 10.1.10.0/24, 10.1.30.0/24 } tcp dport 80 redirect to 3128;
  }
}
```

> As mentioned, here the proxy is on the same machine as the firewall. If it is on a different machine, you can use `dnat to <IP>:<PORT>` instead of the `redirect to <PORT>` syntax.
{.is-info}

> Make sure to redirect the packets to the `intercept` listener!
{.is-warning}
