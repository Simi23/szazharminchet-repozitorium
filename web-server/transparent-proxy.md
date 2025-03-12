---
title: Squid Transparent Proxy
description: Squid Transparent Proxy configuration
published: true
date: 2025-03-12T12:13:50.098Z
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

## Basic configuration

The existing configuration file contains the whole documentation. You should create a backup of it, then edit a new file.

For a simple HTTP transparent proxy, you need the following configuration.

```ini
http_access allow all
http_port 3128 intercept
http_port 3129
```

> For the transparent mechanism, you need the `http_port` line with the `intercept` option. However, you still need the regular proxy service to listen as well, otherwise the service will not start.
{.is-warning}

## Modifying headers

You can also add headers to the HTTP responses:

```ini
reply_header_add x-secured-by "clearsky-proxy"
```

## Filtering

Filtering requests can be achieved with the [acl](https://www.squid-cache.org/Doc/config/acl/) and [http_access](https://www.squid-cache.org/Doc/config/http_access/) directives. For example:

```ini
acl CONNECT method CONNECT
acl blockdomains dstdomain .index.hu .fidesz.hu

http_access deny blockdomains CONNECT
http_reply_access deny blockdomains
http_access allow all
```

This configuration blocks the two mentioned domains and allows everything else. The `CONNECT` method is used when the browser knows about the proxy.

## Custom error pages

Error pages are stored at <kbd>/usr/share/squid/errors/&lt;LANGUAGE&gt;/</kbd> by default. You can change these to your liking.

You can also create custom [deny info](https://www.squid-cache.org/Doc/config/deny_info/) pages based on the ACL that was matched to deny the request.

For example, this would display the file CUSTOM_ERROR when the request was denied based on the `blockdomains` acl.

```ini
deny_info CUSTOM_ERROR blockdomains
```

## Miscellaneous config

You can set shutdown lifetime to avoid waiting 30 seconds when the service stops:

```ini
shutdown_lifetime 5 seconds
```

# Firewall setup

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

# TLS Inspection

## Installation

To inspect TLS-encrypted traffic, you need the version of squid compiled with `--with-openssl` and `--enable-ssl-crtd` options. You can get it from the package manager.

```bash
# Make sure squid without OpenSSL is not installed
apt remove squid

apt install squid-openssl
```

## Certificates

When inspecting **TLS-encrypted traffic**, the original server sends their certificate, which is read by the proxy to get the **subject name**. Then, the proxy uses their own CA certificate to **create a new certificate** on-the-fly with the **same subject name** and alterate subject name **as the original certificate**. This way, if the client trusts the proxy CA certificate, the browser will not show any errors regarding broken trust chains or invalid certificates.

**Start by creating said CA certificate.**

```bash
openssl req \
  -new \
  -newkey rsa:4096 \
  -days 365 \
  -nodes \
  -x509 \
  -keyout proxy.key \
  -out proxy.crt \
```

**Also create Diffie-Hellman parameters for the server.**

```bash
openssl dhparam -out dhparam.pem 2048
```

Place these files in the **squid config directory, assign owner and set permissions**. On Debian systems, squid uses the `proxy` user and group. (Other distributions may use `squid`.)

```bash
mkdir /etc/squid/cert
cp proxy.crt proxy.key dhparam.pem /etc/squid/cert/

chown -R proxy:proxy /etc/squid/cert
chmod -R 400 /etc/squid/cert
```

The certificates are now secured and ready to be used by squid.

The certificates which will be created on-the-fly will be stored in a database, which **needs to be initialized** before use. Also set the owner.

```bash
/usr/lib/squid/security_file_certgen -c -s /var/lib/ssl_db -M 4MB

chown -R proxy:proxy /var/lib/ssl_db
```

## Configuration

**Edit** the configuration file <kbd>/etc/squid/squid.conf</kbd>

```bash
http_port 3128
http_port 3129 intercept

# Write this as one line
https_port 3130 intercept ssl-bump \
  generate-host-certificates=on \
  dynamic_cert_mem_cache_size=4MB \
  tls-cert=/etc/squid/cert/proxy.crt \
  tls-key=/etc/squid/cert/proxy.key \
  tls-dh=/etc/squid/cert/dhparam.pem \
  tls-default-ca=on \
  options=NO_SSLv3,NO_TLSv1,NO_TLSv1_1

http_access allow all

sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/lib/ssl_db -M 4MB

acl step1 at_step SslBump1

ssl_bump peek step1
ssl_bump bump all
```

**Restart** the squid service for the new configuration to take effect.

```bash
service squid restart
```

**A few things to note:**

- The firewall needs to redirect **HTTP** (port 80) traffic **to port 3129**, and redirect **HTTPS** (port 443) traffic **to port 3130**.
- The **client browser** will need to **trust the CA certificate** used by **squid**, in this case <kbd>/etc/squid/cert/proxy.crt</kbd>
- The option `tls-default-ca=on` means that squid will **trust the system CA certificates**. This is **important if you use custom certificates** and store them in <kbd>/usr/local/share/ca-certificates</kbd> because this option is **disabled by default**. Sites that squid doesn't trust will not be signed correctly and **display an error in the browser**.
- When using an **intermediate CA** for the proxy, the file referenced with `tsl-cert=` can contain the **whole trust chain**, which will be sent to the client, so it can verify trust using only the root CA certificate.