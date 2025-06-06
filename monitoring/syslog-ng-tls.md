---
title: Syslog-NG with TLS
description: Gathering, and placing logs from remote servers to one place with Syslog-NG (secured)
published: true
date: 2025-06-06T08:10:50.314Z
tags: linux
editor: markdown
dateCreated: 2025-06-06T07:26:45.502Z
---

# Syslog-NG with TLS

## Basics

> You have to install `syslog-ng` package to use the daemon.
> There are some predefined **sources**, **filters** and **destinations** in `/etc/syslog-ng/syslog-ng.conf` file.
> For modularity and seperation, if you create a new config, then please place it under `/etc/syslog-ng/conf.d` directory and name it `*.conf`, because the main configuartion file will include these config files.
{.is-info}

### Source
> The sources of logging. The default source where you get all local system logs is `s_src`.
> Later we will define sources in these to achieve log collection from syslog clients.
{.is-info}

#### Syntax
```
source s_name {
	system();
};
```

### Filter
> A filter you can define when you want to place different daemon logs to different files.
{.is-info}

#### Syntax
```
filter f_name {
	level(info) and /\ or /\ not facility() and /\ or /\ not program();
};
```

### Destination
> The destination where you place, or where you send your logs. On servers you will define files, and on client you will define mainly transport options.
{.is-info}


> If you want to place every log into a tty line, you can define a destination to `/dev/ttyX` where **X** means the tty's number.
{.is-success}

#### Syntax
```
destination d_name {
	file("/dev/tty5");
};
```

### Log
> If you want a new log into your server, than you have to use the `log` keyword. In a `log` field you will use one or more (or zero) predefined **source**, **filter** and **destination**
{.is-info}

#### Syntax
```
log{
	source(s_name);
  filter(f_name); # Optional
  destination(d_name);
};
```

### IETF
> One protocol from the two with you can send and receive syslogs. You have to use the `syslog` keyword to use this. In a `syslog` field you will use one or more (or zero) predefined **source**, **filter** and **destination**. You have to define this in a source or a destination.
{.is-info}

#### Syntax
```
syslog{
	ip-protocol(4) # If you define 6 it wil listen on IPv6 and IPv4 too.
    port(6514) # Number between 1-65536
    transport("tls") # udp,tcp,tls
    tls (
    	cert-file("/ca/SRV.pem")
      key-file("/ca/SRV.key")
      ca-file("/ca/CA.crt")
      ca-dir("/ca/")
      # peer-verify(optional-untrusted); # You can define this, there will be a table under this what will provide which option do what.
};
```
### IETF
> One protocol from the two with you can send and receive syslogs. You have to use the `syslog` keyword to use this. In a `syslog` field you will use one or more (or zero) predefined **source**, **filter** and **destination**
{.is-info}

#### Syntax
```
syslog{
	source(s_name);
  filter(f_name); # Optional
  destination(d_name);
};
```


### BSD
> One protocol from the two with you can send and receive syslogs. You have to use the `network` keyword to use this. In a `network` field you will use one or more (or zero) predefined **source**, **filter** and **destination**. You have to define this in a source or a destination.
{.is-info}

#### Syntax
```
network{
	ip-protocol(4) # If you define 6 it wil listen on IPv6 and IPv4 too.
    port(6514) # Number between 1-65536
    transport("tls") # udp,tcp,tls
    tls (
    	cert-file("/ca/SRV.pem")
      key-file("/ca/SRV.key")
      ca-file("/ca/CA.crt")
      ca-dir("/ca/")
      # peer-verify(optional-untrusted); # You can define this, there will be a table under this what will provide which option do what.
};
```

### Peer-verify()
> The deafult value is **required-trusted**.
{.is-info}

| Option             | No cert             | Invalid cert        | Valid cert      |
| ------------------ | ------------------- | -------------       | -----------            |
| optional-untrusted | TLS-encryption      | TLS-encryption      | TLS-encryption |
| optional-trusted   | TLS-encryption      | rejected connection | TLS-encryption |
| require-untrusted  | rejected connection | TLS-encryption      | TLS-encryption |
| require-trusted    | rejected connection | rejected connection | TLS-encryption |


## Transport over TLS
> You have to generate [x509 certificates](/cert/openssl). I recommend to create chains from certificates.
> After the creation make the devices trust the CA certificate, and start to configure your devices.
> [HERE](https://syslog-ng.github.io/admin-guide/100_TLS-encrypted_message_transfer/004_TLS_options) you can check every TLS option.
{.is-info}

### Server side configuration

```
source s_dhcp {
	syslog(
  	ip-protocol(4)
    port(6514)
    transport("tls")
    tls (
    	cert-file("/ca/SRV.pem")
      key-file("/ca/SRV.key")
      ca-file("/ca/CA.crt")
      ca-dir("/ca/")
    )
  );
};

destination d_dhcp {
	file("/log/dhcp.log");
};

log{
	source(s_dhcp);
  destination(d_dhcp);
};
```

### Client side configuration

```
destination d_dhcp{
	syslog(
  	"SRV.lego.dk"
    	port(6514)
      transport("tls")
      tls(
      	cert-file("/ca/CLT.pem")
        key-file("/ca/CLT.key")
        ca-file("/ca/CA.crt")
      )
  );
};

filter f_dhcp{
	program("dhcpd") or program("dhclient");
};

log {
	source(s_src);
  filter(f_dhcp);
  destination(d_dhcp);
  
  # ( or if you want to send everything except your filter)
  # filter { 
  #	  not filter(f_dhcp)
  # };
};
```
