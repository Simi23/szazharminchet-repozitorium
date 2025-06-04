---
title: OpenSSL Certificate Revocation
description: Creating Certificate Revocation Lists (CRL) and publishing them via CRL Distribution Points (CDP)
published: true
date: 2025-06-04T13:54:55.954Z
tags: linux
editor: markdown
dateCreated: 2025-06-04T13:54:55.954Z
---

# Creating the revocation list

First, create a directory structure for the certificates.

```bash
mkdir -p /ca/crl
```

Now, generate a new Root CA certificate.

```bash
openssl genrsa -aes256 -out CA.key 4096

openssl req -x509 -new -nodes \
  -key CA.key -sha256 \
  -days 7200 \
  -out CA.crt \
  -subj '/C=DK/O=Lego/CN=Lego Root CA'
```

Next, create a CRL index file.

```bash
touch /ca/crl/index.txt
```

Create a file for the CRL number. It should contain only `00`.

```bash
echo 00 > /ca/crl/crl_number
```

Create the file <kbd>/ca/crl_openssl.conf</kbd> with the following contents.

```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
database = /ca/crl/index.txt
crlnumber = /ca/crl/crl_number

default_days = 365
default_crl_days = 30
default_md = default
preserve = no

[ crl_ext ]
authorityKeyIdentifier = keyid:always,issuer:always
```

Now create the CRL itself:

```bash
openssl ca -gencrl \
  -keyfile /ca/CA.key \
  -cert /ca/CA.crt \
  -out /ca/crl/lego_crl.pem \
  -config /ca/crl_openssl.conf
```

# Publishing the revocation list

To publish the revocation list in a CDP (CRL Distribution Point), you need to extend the x509v3 extensions of the certificates signed by your CA. It should contain the following field:

```ini
...
crlDistributionPoints=URI:http://crl.lego.dk/lego_crl.pem
...
```

This assumes you have a web server running with the `lego_crl.pem` file present, reachable at `crl.lego.dk`.

> For distributing CRLs, only use HTTP, not HTTPS!
{.is-warning}

# Revoking certificates

> This is untested
{.is-danger}

To revoke a certificate, you need the file itself. In this example, the certificate to be revoked will be `client.crt`.

```bash
openssl ca -revoke /ca/client.crt \
  -keyfile /ca/CA.key \
  -cert /ca/CA.crt \
  -config /ca/crl_openssl.conf
```

Now, regenerate the CRL (using the same command as before).

```bash
openssl ca -gencrl \
  -keyfile /ca/CA.key \
  -cert /ca/CA.crt \
  -out /ca/crl/lego_crl.pem \
  -config /ca/crl_openssl.conf
```

After publishing the new CRL to your web server, (or any other distribution method) you can test the CRL with the following commands:

```bash
cat /ca/CA.crt /ca/crl/lego_crl.pem > /tmp/test.pem

openssl verify -extended_crl -verbose \
  -CAfile /tmp/test.pem \
  -crl_check /ca/client.crt
```
