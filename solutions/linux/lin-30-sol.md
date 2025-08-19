---
title: ES25 - ModA - 30% Solution
description: 
published: true
date: 2025-08-19T12:07:57.798Z
tags: 
editor: markdown
dateCreated: 2025-08-19T12:07:57.798Z
---

# ES25 - ModA - 30% Solution

## Cowsay

```bash
apt install cowsay

/usr/games/cowsay \
  -f /usr/share/cowsay/cows/xy.cow \
  "Text to print" > /var/www/html/index.html

echo "export ANSIBLE_COW_SELECTION=random" >> ~/.bashrc
source ~/.bashrc
```

## Extra subnets for PAT

Add extra subnet to Loopback interface. Configure SNAT to use the subnet.

```c
chain postrouting {
  ...
  ip saddr 0.0.0.0/0 oif ens65535 snat to 1.1.1.0/24
}
```

## Dynamic routing

### S2S Tunnel

```bash
apt install frr
nano /etc/frr/daemons
vtysh
# cisco, ez win (gergő megoldja)
```

### Public network

```bash
apt install frr
nano /etc/frr/daemons
vtysh
# cisco, ez win (gergő megoldja)
```

## ZFS

```bash
apt install linux-headers-amd64 zfsutils-linux

ls -la /dev/disk/by-path/ | rev | cut -f 3 -d " " | rev >> ./zfs.sh

cat <<EOF >>zfs.sh
zpool create medence \
  -O encryption=aes-256-gcm \
  -O keyformat=passphrase \
  raidz1 \
    <disk-ids...>
EOF

chmod +x ./zfs.sh

# Delete unwanted disks
nano zfs.sh

./zfs.sh

mkdir /share
zfs create medence/uszogumi -o mountpoint=/share
mkdir -p /share/users

zfs mount -l -a

echo "Passw0rd!" > /etc/passphrase
chmod 600 /etc/passphrase

cat <<EOF > /etc/rc.local
#!/bin/bash
zfs mount -l -a < /etc/passphrase
EOF

chmod +x /etc/rc.local
```

## Backup

```bash
#!/bin/bash

# Variables
date=$(date '+%Y-%m-%d_%H:%M:%S')
backupPath="/tmp/$date.tar.gz"

# Logic
tar -cvzf $backupPath /share/users
scp $backupPath root@HQ-SAM-2:/backup/users
rm $backupPath
```

Add script to crontab.

## Email record verification

- [Security DNS records](/mail/dns-records)
- [Verification](/mail/verification)

## Email SRV records

```
_submission._tcp.lego.dk	SRV	10 0 587 mail.lego.dk
_smtps._tcp.lego.dk	SRV	10 0 465 mail.lego.dk

_imap._tcp.lego.dk	SRV	10 0 143 mail.lego.dk
_imaps._tcp.lego.dk	SRV	10 0 993 mail.lego.dk
```

## Zabbix access URL

Add port **8080** to <kbd>/etc/apache2/ports.conf</kbd>

<kbd>/etc/apache2/conf-available/zabbix-frontend.conf</kbd> - edit alias to DocumentRoot.

## DDNS

```bash
tsig-keygen DDNS > /etc/bind/ddns.key

cat ddns.key >> /etc/dhcp/dhcpd.conf
```

Edit <kbd>/etc/dhcp/dhcpd.conf</kbd>
```powershell
...

ddns-update-style standard;
update-static-leases on;

subnet 10.0.0.0 netmask 255.255.255.0 {
  ...
  ddns-domainname "domain.com";
  zone domain.com {
    primary 10.1.1.1; # DNS Server
    key "DDNS";
  }
  zone 10.in-addr.arpa {
    primary 10.1.1.1; # DNS Server
    key "DDNS";
  }
}
```

Edit <kbd>/etc/bind/named.conf.local</kbd>

```powershell
...
include "/etc/bind/ddns.key"
...

zone domain.com {
  ...
  allow-update { key DDNS; };
  ...
};
```

## Syslog without client cert auth

```
...
tls(
  peer-verify(optional-untrusted)
)
```

## IPv6 tunnel

<kbd>/etc/network/interfaces</kbd>
```
iface ipsec0 inet static
  ...
  pre-up ip tunnel add ipsec0 mode ip6gre remote 2001::xy local 2001:yz
  ...
```

## LDAP Replication

[LDAP guide](/directory-services/openldap#replication)

