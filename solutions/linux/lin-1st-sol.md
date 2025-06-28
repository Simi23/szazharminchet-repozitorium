---
title: ES25 - ModA - 1st Solution
description: 
published: true
date: 2025-06-28T08:47:42.124Z
tags: linux, es25, es25-linux
editor: markdown
dateCreated: 2025-06-28T08:18:12.032Z
---

# ES25 - ModB - 1st Solution

[//]: <> (General)
<details>
<summary>General</summary>

- Hostname (`Rename-Computer -Name HOSTNAME`)
- Network configuration
- Time Zone
- Keyboard layout
- NTP
  
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
sysLocation YOURSYSTEMLOCATION
sysContact NAME email@address

agentaddress 0.0.0.0, [::]

createuser Administrator MD5 "Passw0rd!" AES256C "Passw0rd!"
rouser Administrator authpriv  
```  
4. Ldap login as on CLT
5. SMB Share automount
6. Default webserver (TLS)
  
</details>

[//]: <> (Backup)
<details>
<summary>Backup</summary>

  
</details>

[//]: <> (CA)
<details>
<summary>CA</summary>

  
</details>


[//]: <> (CLT)
<details>
<summary>CLT</summary>

  
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

  
</details>

[//]: <> (S2S IPSec)
<details>
<summary>S2S IPSec</summary>

  
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

