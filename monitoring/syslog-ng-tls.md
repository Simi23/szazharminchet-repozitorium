---
title: Syslog-NG with TLS
description: Gathering, and placing logs from remote servers to one place with Syslog-NG (secured)
published: true
date: 2025-06-06T07:41:59.574Z
tags: linux
editor: markdown
dateCreated: 2025-06-06T07:26:45.502Z
---

# Syslog-NG with TLS

## Basics

> You have to install `syslog-ng` package to use the daemon.
> There are some predefined **sources**, **filters** and **destinations** in `/etc/syslog-ng/syslog-ng.conf` file.
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
> One protocol from the two with you can send and receive syslogs. You have to use the `network` keyword to use this. In a `network` field you will use one or more (or zero) predefined **source**, **filter** and **destination**
{.is-info}

#### Syntax
```
network{
	source(s_name);
  filter(f_name); # Optional
  destination(d_name);
};
```
