---
title: isc-dhcp-server
description: Isc-dhcp-server configuration. DDNS, high availability included.
published: true
date: 2025-02-17T10:58:18.606Z
tags: linux
editor: markdown
dateCreated: 2025-02-15T10:02:40.962Z
---

# ISC-DHCP-SERVER

## Installation

Install isc-dhcp-server server, it will show errors at the installation, but we will resolve the issues. This config is for high availability DHCP servers.

```shell
apt install isc-dhcp-server
```

## Configuration

Edit the `/etc/default/isc-dhcp-server` file, if you don't know which interface you want to use just check it out with `ip address` command

```cfg
...

INTERFACESv4="ens33"
INTERFACESv6="ens33"
```

### Primary server

Now you can set everything you need in these to config files: `/etc/dhcp/dhcpd.conf` `/etc/dhcp/dhcpd6.conf`. These files have many sample configurations. Uncomment the one that matches your needs, and edit the details. Here is a sample default configuration. You have to generate a key, you can do it by using `tsig-keygen <keyname>`. You have to use this key more times so save it for the use on secondary server.

```cfg
key "omapi-key" {
        algorithm hmac-sha256;
        secret "4iTnXeMsp7Z/qRhtr6dz5ISIX0byYWgWulA6pgw>
};

omapi-port 7911;
omapi-key omapi-key;

failover peer "failover" {
        primary;
        address 10.0.0.1;
        peer address 10.0.0.2;
        mclt 3600;
        split 128;
        load balance max seconds 3;
}

subnet 10.0.0.0 netmask 255.255.255.0 {
  option routers 10.0.0.254;
  option domain-name-servers 10.0.0.1, 10.0.0.2;

  pool {
    range 10.0.0.10 10.0.0.250;
    failover peer "failover";
  }
}
```

Restart the daemon:
```
systemctl restart isc-dhcp-server
```

### Secondary server

Now you can set everything you need in these to config files: `/etc/dhcp/dhcpd.conf` `/etc/dhcp/dhcpd6.conf`. These files have many sample configurations. Uncomment the one that matches your needs, and edit the details. Here is a sample default configuration. You have to generate a key, you can do it by using `tsig-keygen <keyname>`. You have to use this key more times so save it for the use on secondary server.

```cfg
key "omapi-key" {
        algorithm hmac-sha256;
        secret "4iTnXeMsp7Z/qRhtr6dz5ISIX0byYWgWulA6pgw>
};

omapi-port 7911;
omapi-key omapi-key;

failover peer "failover" {
        secondary;
        address 10.0.0.2;
        peer address 10.0.0.1;
        load balance max seconds 3;
}

subnet 10.0.0.0 netmask 255.255.255.0 {
  option routers 10.0.0.254;
  option domain-name-servers 10.0.0.1, 10.0.0.2;

  pool {
    range 10.0.0.10 10.0.0.250;
    failover peer "failover";
  }
}
```

Restart the daemon:
```
systemctl restart isc-dhcp-server
```

## DDNS

You can configure DDNS in isc-dhcp-server. You have to do the configurations in the DNS config too, for [Bind9](/DNS/Bind9) in this tutorial you will find what you have to do. You will have to use the same key on DNS and DHCP side on both server.

Add these lines to `/etc/dhcp/dhcpd.conf`

```
...

include "/etc/bind/DHCP.key";

zone company.com. {
  primary 10.0.0.1;
  key dhcp-key;
}

zone db.10.0.0. {
  primary 10.0.0.1;
  key dhcp-key;
}

...

subnet 10.0.0.0 netmask 255.255.255.0 {
  ...
  
  ddns-domainname "company.com";
  ddns-rev-domainname "10.0.0";
}
```

Restart the daemon.

```
systemctl restart isc-dhcp-server
```

