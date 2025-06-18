---
title: Strongswan Route-based VPN
description: Set up a GRE-over-IPSec tunnel with Strongswan
published: true
date: 2025-06-18T09:25:19.255Z
tags: linux
editor: markdown
dateCreated: 2025-06-18T09:25:19.255Z
---

# Installation

First, install Strongswan. This guide will use the new, swanctl-based configuration.

```bash
apt install charon-systemd strongswan-swanctl
```

# Tunnel setup

In order to set up GRE tunnels in Linux, you can use the **ip** utility command. However, tunnels created by using the command are **not persistent**. A better solution is to create a new interface in <kbd>/etc/network/interfaces</kbd> and utilize *up/down* scripts to create the tunnel every time the interface is brought up. This way you can also set the IP address of the tunnel interface.

Add these lines to <kbd>/etc/network/interfaces</kbd>:

```ini
auto ipsec0
iface ipsec0 inet static
  address 10.200.0.1/30
  pre-up ip tunnel add ipsec0 mode gre local 1.1.1.1 remote 2.2.2.2
  up ip link set ipsec0 up
  down ip link set ipsec0 down
  post-down ip tunnel del ipsec0
```

After restarting the **networking** service, the GRE tunnel itself should already work - *although, without encryption.*

```bash
service networking restart
```

If the tunnel is up, you can ping the IP address of the other host's tunnel interface. To reach remote networks behind these hosts, you need routes installed in the kernel pointing to the other host's tunnel IP. *(This will be covered in the IPSec section.)*

# Strongswan setup

Now that the GRE tunnel is working, encryption should be configured. This guide uses certificates for authentication, which will not be covered here. *(You can find information about it [here](/vpn/linux-windows-strongswan-cert-new).)*

Edit <kbd>/etc/swanctl/swanctl.conf</kbd>.

```
connections {
  s2s {
    local_addrs = 1.1.1.1
    remote_addrs = server2.com
    
    local {
      auth = pubkey
      certs = server1.crt
    }
    remote {
      auth = pubkey
      id = "C=HU, O=Test Ltd., CN=server2.com"
    }
    children {
      net {
        local_ts = dynamic[gre]
        remote_ts = dynamic[gre]
        mode = transport
        start_action = trap|start
        updown = /fw/updown.sh
      }
    }
    version = 2
  }
}
```

The updown script will contain the installation of routes. Create the file <kbd>/fw/updown.sh</kbd> and edit it. Make sure it is executable.

```bash
mkdir /fw
touch /fw/updown.sh
chmod +x /fw/updown.sh
nano /fw/updown.sh
```

```bash
#!/bin/bash

if [ $PLUTO_VERB == "up-host" ]; then
  ip route add 20.0.0.0/16 via 10.200.0.2 dev ipsec0
fi
if [ $PLUTO_VERB == "down-host" ]; then
  ip route del 20.0.0.0/16 via 10.200.0.2 dev ipsec0
fi
```

Now, reload all configuration and restart strongswan:

```bash
swanctl --load-all
service strongswan restart
```

If you look at the logs of the `strongswan` unit, you should see that the IKE SA is established.