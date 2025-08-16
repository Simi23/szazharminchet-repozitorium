---
title: ES25 - ModA - 1st Solution
description: 
published: true
date: 2025-08-16T14:19:33.084Z
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
- Create `/ca` directory
  
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
  
Add cronjob that runs every five minute and place all output to `/dev/null`
</details>

[//]: <> (CA)
<details>
<summary>CA</summary>

  **First, create the CA and SUBCA certificates.**
  
  ```bash
  # Create the CA key and certificate
  openssl genrsa -aes256 -out CA.key 4096
  openssl req -new -nodes -x509 \
    -key CA.key -sha256 \
    -out CA.crt \
    -days 7200 \
    -subj '/C=DK/O=Lego APS/CN=Lego APS Root CA'
  
  # Create the SUBCA
  ## Create the v3 extension file for the SUBCA
  cat <<EOF > SUBCA.v3.ext
  subjectKeyIdentifier=hash
  authorityKeyIdentifier=keyid:always,issuer
  basicConstraints=CA:TRUE
  crlDistributionPoints=URI:http://crl.lego.dk/root_crl.pem
  EOF
  
  ## Now create a request for the SUBCA and sign it with the CA
  openssl req -new -nodes \
    -newkey rsa:4096 \
    -keyout SUBCA.key \
    -out SUBCA.csr \
    -subj '/C=DK/O=Lego APS/CN=Lego APS Intermediate CA'
  
  openssl x509 -req -CA CA.crt -CAkey CA.key -CAcreateserial \
    -in SUBCA.csr -out SUBCA.crt -days 3650 -sha256 \
    -extfile SUBCA.v3.ext
  ```
  
  **Now, create directories for all hosts which need certificates. After that, create a base v3 extension file like this:**
  
  ```ini
  authorityKeyIdentifier=keyid,issuer
  basicConstraints=CA:FALSE
  keyUsage=keyEncipherment,dataEncipherment,digitalSignature,nonRepudiation
  crlDistributionPoints=URI:http://crl.lego.dk/sub_crl.pem
  subjectAltName=@alt_names
  
  [alt_names]
  DNS.1=
  IP.1=
  email.1=
  ```
  
  **After that, distribute this to all directories and fill them accordingly. Then, use the following script to create and sign all certificates.**
  
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
  
After the installation is done, restart the computer and use these commands to be able to configure it using Ansible.
  
  ```bash
adduser ansible # You will have some dialog, give it password and spam ENTER
  
echo "ansible	ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers
  ```
  
</details>

[//]: <> (CRL)
<details>
<summary>CRL</summary>

  After the certificates are already done, create some files necessary for the CRL.
  
  ```bash
  cd /ca
  mkdir crl
  touch crl/index.txt
  echo "00" > crl/crlnumber
  ```
  
  Now, create <kbd>/ca/crl_openssl.conf</kbd> with the following contents:
  
  ```ini
  [ca]
  default_ca = CA_default
  
  [CA_default]
  database = /ca/crl/index.txt
  crlnumber = /ca/crl/crlnumber
  default_days = 365
  default_crl_days = 30
  default_md = default
  preserve = no
  
  [crl_ext]
  authorityKeyIdentifier=issuer:always,keyid:always
  ```
  
  Now, generate the CRL with the following command:
  
  ```bash
  openssl ca -gencrl \
    -keyfile /ca/CA.key \
    -cert /ca/CA.crt \
    -out /ca/crl/root_crl.pem \
    -config /ca/crl_openssl.conf
  ```
  
  Do these steps for both the CA and the SUBCA.
  
  Use `lsyncd` to sync the crl to the webserver it is hosted on. Don't forget **ssh-copy-id**.
  
</details>

[//]: <> (DHCP)
<details>
<summary>DHCP</summary>
- Generate a key on DNS server and copy it to /etc/dhcp/
- Edit /etc/default/isc-dhcp-server add the interface to IPv4
- Copy the content of key to the end of /etc/dhcp/dhcpd.conf
- Create a failover peer:
```cfg
failover peer "fail" {
	  primary;
  	address 10.1.10.21;
  	peer address 10.1.10.22;
  	mclt 3600;
  	split 128;
  	load balance max seconds 3;
}
```
- Edit the configuration for your needs.
- Copy the files to HQ-SAM-2 and BR-SRV. Adjust failover settings on HQ-SAM-2, and adjust scope settings on BR-SRV!
  
Install **isc-dhcp-relay** role to **R-BR** and **R-HQ**.
Make sure the delimiter is a space in the config! Not ',' or ';'
</details>


[//]: <> (DNS)
<details>
<summary>DNS</summary>
Create all views! Don't forget to create every record and don't forget to add SPF and DKIM
SPF: @ IN TXT "v=spf1 a mx -all"
DMARC: _dmarc IN TXT "v=DMARC1,p=quarantine"
SRV: _submission._tcp.mail.lego.dk SRV 10 0 587 mail.lego.dk
SRV: _imaps._tcp.mail.lego.dk SRV 10 0 993 mail.lego.dk
</details>

[//]: <> (Email + DKIM)
<details>
<summary>Email + DKIM</summary>

  **Usual config:**

  - [SMTP/IMAP **separate** server](/mail/smtp-imap)
  - [SMTP/IMAP **single** server](/mail/smtp-imap-single-server)
  - [SPF, DKIM, DMARC **setup**](/mail/dns-records)
  - [SPF, DKIM, DMARC **verification**](/mail/verification)
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
Install **FileZilla**
Add a new host where you enter the credentials of *carl*, the destination as *BR-SRV.herning.lego.dk*, and the main directory too wwwroot so use */var/www/html*
  
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
Install following packages
```bash
apt install \
  apache2 libapache2-mod-php php-mysql \
  zabbix-server-mysql zabbix-frontend-php zabbix-agent zabbix-web-service \
  snmp
```
  
- Edit configuration file, include database password!
- Create a copy of /usr/share/doc/zabbix-mysql/README.debian, make it executeable and get rid off not neccessary lines.
- Run the script
- Create /etc/zabbix/alert.d/log.sh and give it execute permission 
```bash
#!/bin/bash
date=$(date '+%Y-%m-%d_%H:%M:%S')
echo "[$date] - $1 - $2" >> $3
```
  
Enable Zabbix, and make the replace the Alias with docuemnt root in the config file.
  
Restart apache2 and zabbix-server services.
</details>

[//]: <> (ZFS)
<details>
<summary>ZFS</summary>
```bash
apt install linux-headers-amd64 zfsutils-linux
```

Create a script that will create your ZFS pool:
```
mkdir /scripts
touch /scripts/zfs.sh
chmod +x /scripts/zfs.sh
ls -l /dev/disks/by-path/ | rev | cut -d' ' -f 3 | rev >> /scripts/zfs.sh
```
  
Edit /scripts/zfs.sh
```sh
	zpool create pool \
  	-O encryption="aes-256-gcm" \
  	-O keyformat="passphrase" \
  		raidz1 \
  			Put here the names of disks
```
  
Create a pool
```bash
mkdir /share
zfs create -o mountpoint=/share pool/share
mkdir /share/users
```
  
Make it auto mountable
```bash
echo "Passw0rd!" > /etc/passphrase
chown root:root /etc/passphrase
chmod 600 /etc/passphrase
echo "#!/bin/bash" > /etc/rc.local
echo "zfs mount -l -la < /etc/passphrase" >> /etc/rc.local
chmod +x /etc/rc.local
  
</details>


[//]: <> (Web)
<details>
<summary>Web</summary>
```bash
apt install cowsay
```
  
</details>



