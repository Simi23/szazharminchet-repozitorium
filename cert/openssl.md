---
title: Certification Authority
description: x509 Certification Authority setup with openssl
published: true
date: 2025-02-12T20:59:29.513Z
tags: linux
editor: markdown
dateCreated: 2025-02-11T11:13:16.841Z
---

# Root Certification Authority
Create a root CA to sign all your certificates.

This guide will use the following names:
 - Company name: `My Org`
 - Root CA name: `My Org Root CA`
 - Country code: `HU`

Create the CA:

```bash
mkdir /ca && cd /ca

# Generate CA key
openssl genrsa -aes256 -out CA.key 4096
# Generate CA certificate
openssl req -x509 -new -nodes \
  -key CA.key -sha256 \
  -days 365 \
  -out CA.crt \
  -subj '/C=HU/O=My Org/CN=My Org Root CA'
```

Add CA certificate to trusted CA certificates on a machine. This is needed for the machine to validate certificates created by this CA.
```sh
cp CA.crt /usr/local/share/ca-certificates/
update-ca-certificates
```

> The `update-ca-certificates` command only reads files with `.crt` extension.
{.is-warning}

# Generate certificates

To generate a certificate, first create a signing request (CSR). This command will also create the key alongside the request.

```bash
openssl req -new -nodes \
  -out CERT.csr \
  -newkey rsa:4096 \
  -keyout CERT.key \
  -subj '/C=HU/O=My Org/CN=<FQDN>'
```

Create and edit the file `CERT.v3.ext`. This will be used to supply the x509v3 extensions for signing the certificate.

```bash
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

# Add any alternative DNS names or IP addresses for the certificate
[alt_names]
DNS.1 = <Domain Address>
IP.1 = <IP Address>
```

Sign the certificate. This will output the certificate file which can be used with the key file.

```bash
openssl x509 -req \
  -in CERT.csr \
  -CA CA.crt \
  -CAkey CA.key \
  -CAcreateserial \
  -out CERT.crt \
  -days 365 \
  -sha256 \
  -extfile CERT.v3.ext
```

If needed, bundle a certificate and its key into a pkcs12 pack:

```bash
openssl pkcs12 -export \
  -inkey CERT.key \
  -in CERT.crt \
  -out CERT.p12
```

# Testing

## Test a service running SSL/TLS

Connect to a service using openssl

```bash
openssl s_client -connect <host>:<port>
```

## Read contents of certificate

```bash
openssl x509 -noout -text -in 'cert.crt'
```