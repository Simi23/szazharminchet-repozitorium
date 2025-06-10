---
title: Strongswan S2S VPN with redirection on tunnel failure
description: NFTables DNAT traffic redirection on tunnel failure to access other site's public services
published: true
date: 2025-06-10T07:07:16.119Z
tags: linux
editor: markdown
dateCreated: 2025-06-06T13:49:51.456Z
---

# Topology

There are two sites in this case, with one router each:

```
[10.1.0.0/16] -- { RTR-A } ----- { RTR-B } -- [10.2.0.0/16]
```

Public IP addresses:

  - **RTR-A:** 100.0.0.1
  - **RTR-B:** 200.0.0.1

# NFTables setup

When the tunnel goes down, all traffic meant to go to the other site will be redirected to the public IP address of the other router. This way, the public services are still accessible over WAN.

To accomplish this, create a ruleset like this: *(This is on RTR-A)*

```c
table inet filter {
  set traffic_reroute {
    typeof ip saddr
    flags interval
  }
  chain srcnat {
    type nat hook postrouting priority srcnat
    ip saddr 10.1.0.0/16 ip daddr != 10.2.0.0/16 oif ens192 masquerade
  }
  chain dstnat {
    type nat hook prerouting priority dstnat
    ip saddr @traffic_reroute ip daddr 10.2.0.0/16 dnat to 200.0.0.1
  }
}
```

The `srcnat` chain handles PAT, so LAN users can access the internet.

The `dstnat` chain will DNAT all IP packets meant to the other site to the public IP of **RTR-B** whenever the `traffic_reroute` set contains the local site's prefix. This will be controlled by the up/down script invoked by Strongswan whenever the IKE SA goes up/down.

# Strongswan setup

This guide uses the swanctl-style config. When the tunnel just fails without a proper IKE SA deletion, the timeout needs to be sped up because the default timeout is about ~160 seconds. To change this, edit /etc/strongswan.conf and add the following lines:

```c
charon {
  ...
  retransmit_tries = 3
  retransmit_timeout = 0.6
  retransmit_base = 1.1
}
```

This will break the tunnel after ~10 seconds of it not working.

Also, edit /etc/swanctl/swanctl.conf and add the following lines:

```c
connections {
  my-connection {
    dpd_delay = 3s
    ...
    children {
      my-child {
        ...
        start_action = trap|start
        dpd_action = clear
        updown = /fw/updown.sh
      }
    }
  }
}
```

Now create the updown script at <kbd>/fw/updown.sh</kbd>:

```bash
#!/bin/bash

if [ $PLUTO_VERB == "up-client" ]; then
  nft delete element inet filter traffic_reroute { 10.1.0.0/16 }
fi

if [ $PLUTO_VERB == "down-client" ]; then
  nft add element inet filter traffic_reroute { 10.2.0.0/16 }
fi
```

Now restart Strongswan:

```bash
systemctl restart strongswan
```