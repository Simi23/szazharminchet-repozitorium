---
title: Local DNS Resolver
description: Configure a local DNS resolver with unbound for advanced DNS setups
published: true
date: 2025-06-18T08:23:28.684Z
tags: linux
editor: markdown
dateCreated: 2025-06-18T08:23:28.684Z
---

# Introduction

This guide will demonstrate the configuration of Unbound DNS resolver for complex DNS setups. In particular, queries of separate domains will be forwarded to separate servers.

# Installation

Install Unbound with the following command:

```bash
apt install unbound
```

# Configuration

Edit <kbd>/etc/unbound/unbound.conf</kbd> and add the following lines:

```yaml
server:
  interface: 0.0.0.0 # Listen on all interfaces
  outgoing-interface: 10.10.10.10 # Use this IP as source for queries
  
  # Send all queries of zone1.com to 1.1.1.1
  forward-zone:
    name: "zone1.com"
    forward-addr: 1.1.1.1
  
  # Send all queries of zone2.com to 2.2.2.2
  forward-zone:
    name: "zone2.com"
    forward-addr: 2.2.2.2
  
  # Send all other queries to 8.8.8.8
  forward-zone:
    name: "."
    forward-addr: 8.8.8.8
```

If the servers you forward to are **not DNSSEC-enabled**, you should disable the root trust anchor by **commenting all lines** in <kbd>/etc/unbound/unbound.conf.d/root-auto-trust-anchor-file.conf</kbd>.

Now, **restart** Unbound to apply settings.

```bash
service unbound restart
```

Edit /etc/resolv.conf so the host uses the Unbound resolver.

```c
nameserver 127.0.0.1
```

# Flushing cache

If you need to flush the cache of Unbound, use the following command:

```bash
unbound-control flush_zone .
```

This will flush the cache of **all zones**. This can be useful in scripting, too.