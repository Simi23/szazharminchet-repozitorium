---
title: ES25 - ModA - 1st Solution
description: 
published: true
date: 2025-08-11T09:39:50.664Z
tags: linux, es25, es25-linux
editor: markdown
dateCreated: 2025-06-28T08:18:12.032Z
---

# ES25 - ModA - 1st Solution

[//]: <> (General)
<details>
<summary>General</summary>

- Hostname
- Network configuration
- Time Zone
- Keyboard layout
- NTP
- SSH Root login
  
</details>

[//]: <> (Ansible)
<details>
<summary>Ansible</summary>

1. SSH Key distrib + General config
2. Syslog over TLS (Pregenerate cert - create record for this in DNS too)
  ```
destination d_dest{
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

log {
	source(s_src);
	destination(d_dest);
};
  ```
3. SNMP oid for CPU load avarage `1.3.6.1.4.1.2021.10.1.3.1`
```
agentaddress 0.0.0.0, [::]

createuser Administrator MD5 "Passw0rd!" AES256C "Passw0rd!"
rouser Administrator authpriv  
```  
4. Ldap login as on CLT
Edit <kbd>/etc/sssd/sssd.conf</kbd>, add:
```
services=nss, pam
```
  Enable auto home directory creation with this command:
```bash
	pam-auth-update --enable mkhomedir
```
5. SMB Share automount
6. Default webserver (TLS)
  DON'T FORGET PAGE CONTENT
  
</details>

[//]: <> (Backup)
<details>
<summary>Backup</summary>
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
  
Add cronjob that runs every five minute and place all output to /dev/null
</details>

[//]: <> (CA)
<details>
<summary>CA</summary>

  ```bash
#!/bin/bash

cd /ca

hostnames=("HQ-DC" "HQ-DMZ-1" "HQ-DMZ-2" "R-HQ" "HOME" "R-BR" "BR-SRV" "BR-CL")
domains=(".billund" ".billund" ".billund" ".billund" "" ".herning" ".herning" ".herning")
ips=("10.1.10.11" "10.1.20.11" "10.1.20.12" "10.1.10.1" "10.1.10.1" "10.200.0.2" "10.2.10.11" "10.2.10.11")

for i in {0..7}; do
        name="${hostnames[$i]}"
        fullname="${hostnames[$i]}${domains[$1]}.lego.dk"
        filename="$name/$name"
        ip="${ips[$i]}"

        openssl req -new -nodes -newkey rsa:4096 -keyout $filename.key -out $filename.csr -subj "/C=DK/O=Lego APS/CN=$fullname"
        openssl x509 -req -CA SUBCA.crt -CAkey SUBCA.key -CAcreateserial -in $filename.csr -out $filename.crt -days 365 -sha256 -extfile $name/base.v3.ext

        sshpass -p "Passw0rd!" ssh-copy-id $ip

        scp CA.crt SUBCA.crt $filename.crt $filename.key $ip:/ca/
done
```
  
</details>


[//]: <> (CLT)
<details>
<summary>CLT</summary>
```bash
apt install -y gnome, thunderbird, filezilla
```
After the installation done restart the computer and use these commands to be able to configure it using Ansible
  
```bash
adduser ansible # You will have some dialog, give it password and spam ENTER
  
echo "ansible	ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers
```
  
</details>

[//]: <> (CRL)
<details>
<summary>CRL</summary>

  
</details>

[//]: <> (DHCP)
<details>
<summary>DHCP</summary>

  
</details>


[//]: <> (DNS)
<details>
<summary>DNS</summary>
Create all views! Don't forget to create every record and don't forget to add SPF and DKIM
SPF: @ IN TXT "v=spf1 a mx -all"
DMARC: _dmarc IN TXT "v=DMARC1,p=quarantine"
SRV: _submission.tcp.mail.lego.dk 10 0 0 255
</details>

[//]: <> (Email + DKIM)
<details>
<summary>Email + DKIM</summary>

  
</details>

[//]: <> (Firewall)
<details>
<summary>Firewall</summary>

  
</details>

[//]: <> (LDAP)
<details>
<summary>LDAP</summary>

  
</details>

[//]: <> (RA VPN)
<details>
<summary>RA VPN</summary>

  
</details>


[//]: <> (Samba)
<details>
<summary>Samba</summary>

  
</details>

[//]: <> (SFTP)
<details>
<summary>SFTP</summary>

  
</details>


[//]: <> (S2S Plain)
<details>
<summary>S2S Plain</summary>

  Add to <kbd>/etc/network/interfaces</kbd>
  
  ```
auto ipsec0
iface ipsec0 inet static
  address 10.200.0.1/24
  pre-up ip tunnel add ipsec0 mode gre local 203.0.113.2 remote 203.0.113.10
  up ip link set ipsec0 up
  down ip link set ipsec0 down
  post-down ip tunnel del ipsec0
```
  
</details>

[//]: <> (S2S IPSec)
<details>
<summary>S2S IPSec</summary>

  Edit <kbd>/etc/strongswan.conf</kbd>
  
  ```c
charon {
  ...
  retransmit_tries = 3
  retransmit_timeout = 0.6
  retransmit_base = 1.1
}
```
  
</details>

[//]: <> (Syslog)
<details>
<summary>Syslog</summary>

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
  
destination d_else {
    file("/log/dump.log");
};

filter f_dhcp{
  program("dhcpd") or program("dhclient");
};
  
log{
  source(s_dhcp);
  filter(f_dhcp);
  destination(d_dhcp);
};
  
log{
  source(s_dhcp);
  filter {
     not filter(f_dhcp)
  };
  destination(d_else);
};
  ```
  
</details>

[//]: <> (Zabbix)
<details>
<summary>Zabbix</summary>

  
</details>

[//]: <> (ZFS)
<details>
<summary>ZFS</summary>

  
</details>


[//]: <> (Web)
<details>
<summary>Web</summary>

  
</details>


