---
title: NFTables
description: NFTables Firewall Common Configurations
published: true
date: 2025-03-10T11:15:55.818Z
tags: linux
editor: markdown
dateCreated: 2025-03-10T11:15:55.818Z
---

# Installation

Debian should include nftables by default, but if it isn't present, you can install it.

```bash
apt install nftables
```

However, it isn't enabled by default, which you can achieve by running:

```bash
systemctl enable nftables.service
```

# Packet flow

To get an understanding of the **packet flow**, and the different **hooks**, study the following diagram.

![nf-hooks.png](/nf-hooks.png)

> For a more complete explanation of the **hooks, priorities and families**, visit the [**nftables wiki**](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks).
{.is-info}

# Configuration

Most likely you will want to use the firewall with multiple interfaces and with packet forwarding. You will need to enable forwarding in <kbd>/etc/sysctl.conf</kbd> by uncommenting these lines:

```bash
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

Apply changes with `sysctl -p`.

The NFTables configuration can be edited with the <kbd>/etc/nftables.conf</kbd> configuration file.

The configuration consists of tables and chains.

## Chains

A regular chain has a definition consisting of a type, a hook, and a priority. You can also define a default policy. (The default value is accept.)

```c
chain my_chain {
	type filter hook forward priority filter;
  policy drop;
}
```

## Matching

There are a lot of ways to match packets, only the most common ones are listed here. For a full description, look at the [nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_headers).

- `ip saddr <IPv4 address / prefix>`
  - Example: `ip saddr {10.1.0.0/24, 10.2.0.0/24}`
- `ip daddr <IPv4 address / prefix>`
- `ip6 saddr <IPv6>`
- `ip6 daddr <IPv6>`
- `tcp sport <PORT>`
- `tcp dport <PORT>`
- `udp sport <PORT>`
- `udp dport <PORT>`
- `iif <INTERFACE_NAME>`
- `oif <INTERFACE_NAME>`

You can also combine matchers.

For a stateful firewall, using the conntrack module is essential. To match packets which already have a state created for them, use:

- `ct state { established, related }`

This is useful to accept return traffic.

## Actions

**Forward/input/output chains:**

- `accept`
- `drop`

---

**NAT chains:**

- `snat to <IP>[:PORT]`
- `masquerade`
- `dnat to <IP>[:PORT]`
- `redirect to <PORT>`

> **Never filter traffic in a NAT chain** because **only the first packet** of the flow will actually hit the chain!
{.is-danger}

---

**Any chain:**

- `counter`: packets which hit this rule will be counted. Use `nft list ruleset` to view all rules along with the counter states.
- `log`: packets which hit this rule will be logged.

> These two actions are **non-final actions**, they always have to be added **before the final action**, e.g. `accept` or `drop`.
{.is-info}

# Example configuration

This is an example NFTables configuration for a stateful firewall.

```bash
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
	chain forward {
  	type filter hook forward priority forward;
    # Apply default drop policy
    policy drop;
    
    # Accept return traffic
    ct state { established, related } accept;
    
    # Allow traffic from INT and DMZ to internet
    ip saddr { 10.1.10.0/24, 10.1.20.0/24 } oif ens18 accept;
    ip6 saddr { 2001:db8:1001:10::/64, 2001:db8:1001:20::/64 } oif ens18 accept;
    
    # Allow traffic from INT to DMZ
    ip saddr 10.1.10.0/24 ip daddr 10.1.20.0/24 accept;
    ip6 saddr 2001:db8:1001:10::/64 ip6 daddr 2001:db8:1001:20::/64 accept;
    
    # Allow traffic from VPN clients to DMZ and INT
    ip saddr 10.1.30.0/24 ip daddr { 10.1.10.0/24, 10.1.20.0/24 } accept;
    ip6 saddr 2001:db8:1001:30::/64 ip6 daddr { 2001:db8:1001:10::/64, 2001:db8:1001:20::/64 } accept;
    
    # Allow mail server to reach LDAP/LDAPS
    ip saddr 10.1.20.10 ip daddr 10.1.10.10 tcp dport { 389,636 } accept;
    ip6 saddr 2001:db8:1001:20::10 ip6 daddr 2001:db8:1001:10::10 tcp dport { 389,636 } accept;
  }
  
  chain srcnat {
  	type nat hook postrouting priority srcnat;
    
    # Configure PAT for INT and DMZ networks
    ip saddr { 10.1.10.0/24, 10.1.20.0/24 } oif ens18 masquerade;
  }
  
  chain dstnat {
  	type nat hook prerouting priority dstnat;
    
    # Create port forwarding rules for external HTTP(S) and DNS traffic
    ip daddr 1.1.1.10 tcp dport { 53,80,443 } dnat to 10.1.20.20;
    ip daddr 1.1.1.10 udp dport 53 dnat to 10.1.20.20;
    
    # Route INT and VPN networks to transparent HTTP proxy running on this host
    ip saddr { 10.1.10.0/24, 10.1.30.0/24 } tcp dport 80 redirect to 3128;
    ip6 saddr { 2001:db8:1001:10::/64, 2001:db8:1001:30::/64 } tcp dport 80 redirect to 3128;
  }
}
```