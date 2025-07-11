---
title: Strongswan Remote Access VPN (EAP-TLS)
description: Strongswan remote access VPN with EAP-TLS authentication and vIP assignment
published: true
date: 2025-07-11T12:14:46.266Z
tags: linux
editor: markdown
dateCreated: 2025-07-11T12:12:49.205Z
---

# Information

This topology uses 3 machines:

```c
|    10.1.1.0/24    |  195.199.203.96/29  |
[RA-SRV] ------- [RA-RTR] -------- [RA-CLT]
| .1           .254 | .100            .97 |
```

Machine roles:

- **RA-SRV** will serve as an inner server, for CA and DNS purposes
- **RA-RTR** is the VPN gateway, the Strongswan server
- **RA-CLT** is the remote access VPN client

# Certificates

The following certificates are created for the machines:

 - ***RA-RTR***
   - **DN:** `C=HU, O=Kontozo, CN=RA-RTR.kontozo.hu`
   - **SubjectAltName:** `IP=195.199.203.100`
 - ***RA-CLT***
   - **DN:** `C=HU, O=Kontozo, CN=remoteaccess@kontozo.hu`
   - **SubjectAltName:** `email=remoteaccess@kontozo.hu`

Copy the right certs and the CA cert to the machines, and make sure it is trusted by the system.

# Server Setup

Install the required packages. This guide will use the new, `swanctl`-style strongswan configuration.

```bash
apt install charon-systemd strongswan-swanctl libcharon-extra-plugins
```

Copy the CA certificate and the server certificate/key to the correct location:

```bash
cp CAcert.pem /etc/swanctl/x509ca/
cp serverCert.pem /etc/swanctl/x509/
cp serverKey.pem /etc/swanctl/private/
```

Edit the strongswan config, `/etc/swanctl/swanctl.conf`.

```c
connections {
  remoteaccess {
    version = 2
    
    local_addrs = 195.199.203.100
    pools = ra_pool

    local {
      auth = eap-tls
      certs = serverCert.pem
    }
    remote {
      auth = eap-tls
    }
    children {
      net {
        local_ts = 10.1.1.0/24
      }
    }
  }
}

pools {
  ra_pool {
    addrs = 10.1.200.0/24
    dns = 10.1.1.1
  }
}

# Include config snippets
include conf.d/*.conf
```

Load the configuration, and monitor the logs for the connection coming from the client.

```bash
service strongswan restart
journalctl -fxeu strongswan
```

# Client Setup

Install the required packages. This guide will use the new, `swanctl`-style strongswan configuration.

```bash
apt install charon-systemd strongswan-swanctl libcharon-extra-plugins
```

Copy the CA certificate and the server certificate/key to the correct location:

```bash
cp CAcert.pem /etc/swanctl/x509ca/
cp clientCert.pem /etc/swanctl/x509/
cp clientKey.pem /etc/swanctl/private/
```

Edit the strongswan config, `/etc/swanctl/swanctl.conf`.

```c
connections {
  remoteaccess {
    version = 2
    
    remote_addrs = 195.199.203.100
    vips = 0.0.0.0
    dpd_delay = 3s

    local {
      auth = eap-tls
      certs = clientCert.pem
    }
    remote {
      auth = eap-tls
      id = "C=HU, O=Kontozo, CN=RA-RTR.kontozo.hu"
    }
    children {
      net {
        remote_ts = 10.1.1.0/24
        start_action = trap|start
        dpd_action = clear
      }
    }
  }
}

# Include config snippets
include conf.d/*.conf
```

Adjust the IPSec retransmit parameters so a broken tunnel is detected sooner. (Which will remove the tunnel and its config, like DNS.)

Edit `/etc/strongswan.conf`

```c
charon {
  ...
  retransmit_base = 1.1
  retransmit_tries = 3
  retransmit_timeout = 2
}

...
```

Load the configuration, and monitor the logs for the connection to the server.

```bash
service strongswan restart
journalctl -fxeu strongswan
```

# Verification

The client should be able to reach the network `10.1.1.0/24`, specified in the traffic selector. When the client reaches that network, the source address of those packets should be one assigned from the pool on the server.
