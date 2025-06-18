---
title: Policy-based Routing in Linux
description: 
published: true
date: 2025-06-18T09:36:55.818Z
tags: linux
editor: markdown
dateCreated: 2025-06-18T09:36:55.818Z
---

# Introduction

**Policy-based routing** on Linux allows influencing the routing decision depending on a number of things, including but not limited to the packets source address or source/destination port, the ingress interface the packet has been received on, the user who originated the packet, or an fwmark value and thereby anything that can be matched by netfilter.

PBR works by specifying rules, which will be evaluated, and route the packets based on different routing tables.

You can see and manage these rules with the `ip rule` command.

# Examples

- Match HTTP(S) traffic and route it using route table 80

```bash
ip rule add dport 80 table 80
ip rule add dport 443 table 80
```

- Match packets by source address

```c
ip rule add from 1.1.1.0/24 table 100
```

- Match by incoming interface

```c
ip rule add iif ens224 table 250
```

Insertion of routes into a routing table is done the same way you would do it with the default table, just add the option `table <number>` to the end of the command.

```c
ip route add default via 2.2.2.2 dev ens22 table 80
ip route add 10.0.0.0/24 via 10.255.0.1 dev ipsec0 table 100
```