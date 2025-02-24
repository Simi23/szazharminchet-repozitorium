# ES25 Wiki

## Certification Authority

Guides related to creating and managing certification authorities, creating certificates.

 - [OpenSSL CA](/cert/openssl): Create a certification authority using OpenSSL
 - [SCEP](/cert/scep): Automatically request/renew certificates from AD CS using a Linux client

## DHCP

- [ISC DHCP Server](/DHCP/isc-dhcp-server)

## Directory services

 - [OpenLDAP](/directory-services/openldap)

## DNS

- [Bind9](/DNS/Bind9)


## Mail Services

 - [SMTP/IMAP](/mail/smtp-imap): Configure a mailing system with Postfix/Dovecot on separate servers, with LDAP authentication backend, and MySQL alias storage.
 - [Roundcube](/mail/roundcube): Install a webmail service on top of your mailing system.

## Virtual Private Networking

### Site-to-site VPN

 - **Strongswan**-**RRAS** with **pre-shared** authentication
   - [`swanctl` config](/vpn/linux-windows-strongswan-new)
   - [`ipsec.conf` config](/vpn/s2s-strongswan-rras-old-psk)
 - **Strongswan**-**RRAS** with **certificate** authentication
   - [`swanctl` config](/vpn/linux-windows-strongswan-cert-new)
   - [`ipsec.conf` config](/vpn/s2s-strongswan-rras-old-cert)

### Remote Access VPN

 - **RRAS server** and **Strongswan client** with **certificate authentication**
   - [`swanctl` config](/vpn/rras-srv-strongswan-ra-client-cert)
   - [`ipsec.conf` config](/vpn/rras-strong-cl-cert-legacy)
 - **Strongswan server** and **Windows built-in IKEv2 client** with **certificate authentication**
   - [`swanctl` config](/vpn/strongswan-srv-windows-client-cert)
   - [`ipsec.conf` config](/vpn/win-clt-strong-srv-cert-legacy)
