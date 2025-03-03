---
title: PKI Server and clients
description: Cisco IOS Certification Authority setup
published: true
date: 2025-03-03T10:08:02.187Z
tags: cisco
editor: markdown
dateCreated: 2025-03-03T10:06:27.938Z
---

# Cisco IOS PKI Server

## Certification Authority setup

Create the CA keypair.

```
crypto key generate rsa label CASRV modulus 4096
```

Enable the builtin HTTP server to allow SCEP to work.

```
ip http server
```

Create the PKI server. Make sure the name matches the label of the keypair you created.

```
crypto pki server CASRV
  database level complete
  grant auto
  cdp-url http://<ip-address>/
  no shutdown
```

> **Notes**
> `<ip-address>` is the IP address of the PKI server.
> 
> `grant auto` is used to accept all requests without checking the SCEP OTP.
> 
> If you want to use CRL checking on the clients, the CDP URL needs to follow this format: `http://<ip-address>/cgi-bin/pkiclient.exe?operation=GetCRL`
> 
> Only issue `no shutdown` after configuring everything else in this block because it locks the configuration of the PKI server. You can issue `shutdown` later to unlock it.
{.is-info}

The server is now ready to accept signing requests through SCEP.

## Certificate client setup

You have to configure the trustpoint.

```
crypto pki trustpoint CASRV
  enrollment url http://<ca-address>:80
  serial-number
  ip-address <ip-address>
  revocation check <crl/none>
  rsakeypair VPN 4096
  auto-enroll 90 regenerate
```

> **Notes**
> The enrollment URL has to point to the PKI server.
> 
> The `serial-number` option will include the device SN in the certificate's subject identifier.
> 
> The `ip-address` option will include an IP address in the subject alternative name.
> 
> Set revocation check as needed.
> 
> The `rsakeypair` option specifies the name and key length of the certificate to generate when enrolling with the authority.
> 
> `auto-enroll 90 regenerate` tells the device to automatically renew its certificate when it reaches 90% of its lifetime.
{.is-info}

After configuring the trustpoint, you have to authenticate it using its fingerprint. Issue the following command and accept the fingerprint.

```
crypto pki authenticate CASRV
```

After the CA is authenticated, enroll a certificate.

```
crypto pki enroll CASRV
```

Now the device will generate a key and receive a certificate. You can view the certificate with the following command.

```
show crypto key mypubkey rsa VPN
```
