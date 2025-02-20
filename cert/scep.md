---
title: SCEP
description: Linux SCEP client obtaining certificates from Windows AD CS
published: true
date: 2025-02-13T10:36:56.858Z
tags: linux, windows
editor: markdown
dateCreated: 2025-02-13T08:42:31.301Z
---

# Windows Setup

Make sure the server is in a domain. Install **AD CS** role. During setup, select **Enterprise CA**, otherwise you won't have access to templates. After the setup is complete, add the feature **Network Device Enrollment Service** to **AD CS** role.

Add the user Administrator *(or any desired user)* to the group **IIS_IUSRS**.

When setting up **AD CS NDES**, select the same user.

Open **Certification Authority** console, right-click **Certificate Templates** and select **Manage**.

Duplicate a template, for example *Web Server* and apply the following settings:

 - In **General**, give a name to the new template, e.g. **NDES**
 - Check *Publish Certificate in Active Directory*
 - In **Request Handling**, check *Allow private key to be exported*
 - In **Cryptography**, select *Requests can use any provider available on the subject's computer*
 - In **Subject Name**, select *Supply in the request*
 - In **Issuance Requirements**, make sure *CA certificate manager approval* is **unchecked**
 - In **Extensions**, select the application policies you need, e.g. *Client Authentication*, *IP security IKE intermediate*, *Secure Email*, *Server Authentication*
 - Check everything in *Key Usage*
 - In **Security** make sure the user selected above has all permissions.

Save the template, exit the template manager. In **Certification Authority** console, right-click **Certificate Templates** and select **New** > **Certificate Template to Issue**. Add the template you just created.

Open **Registry Editor** and navigate to `HKEY_LOCAL_MACHINE/SOFTWARE/Microsoft/Cryptography/MSCEP`. There should be 3 keys ending in *"\*Template"*. Set each of them to the name of the new template, e.g. **NDES**.

Open **IIS Manager** console, and in *Application Pools*, restart *SCEP* application pool. (*Stop*, then *Start*)

# Linux Setup

Install `certmonger`

```bash
apt install certmonger
```

Edit `/etc/certmonger/certmonger.conf` and add:

```
[scep]
challenge_password_otp = yes
```

> This instructs certmonger to delete the one-time password after the first request. This is done because the OTP can only be used once, and while it is not needed for the renewal of the certificate, if you still send it, MSCEP will deny the request.
{.is-info}

Restart certmonger to apply changes.

```bash
service certmonger restart
```

Add your Certification Authority to certmonger.

```bash
getcert add-scep-ca \
  -c WINDOWS \
  -u http://windows.contoso.com/certsrv/mscep
```

What this does:

 - `-c` sets the name of this CA, this will be needed when requesting certificates.
 - `-u` sets the URL of the CA. Replace `windows.contoso.com` with your FQDN/IP

Check the status of the CA.

```bash
getcert list-cas -c WINDOWS
```

If done correctly, the MD5 and SHA1 hash of the SCEP CA certificate should appear.

The SCEP request authentication is based on a one-time password. Each of these can only be used for one certificate, but the renewal process doesn't require one. To obtain a key, navigate to `http://windows.contoso.com/certsrv/mscep`. Log in with the user that you have used during the setup, and copy the OTP.

Create a new request. This is an example for an LDAP cert.

```bash
getcert request \
  -I LDAP \
  -f /testcert/ldap_cert.pem \
  -k /testcert/ldap_key.pem \
  -c WINDOWS \
  -N "CN=srv1.lego.dk" \
  -D ldap.lego.dk \
  -L 7AD4F92FF4FFBB04 \
  -o openldap:openldap \
  -O openldap:openldap \
  -m 600 \
  -C "systemctl restart slapd"
```

What this does:

 - `-I` specifies the request name, this can be used later to reference it.
 - `-f` and `-k` specify the location of the certificate and the key.
 - `-c` references the CA you set up with `getcert add-scep-ca`.
 - `-N` is the **Subject Name** of the request.
 - `-D` can be used to supply a **Subject Alternative Name** in the form of an **FQDN**.
 - `-L` supplies the OTP.
 - `-o` sets the owner of the key file.
 - `-O` sets the owner of the certificate file.
 - `-m` sets the permission of the key file. (`-M` could be used for the certificate file)
 - `-C` specifies a command to run after the certificate is obtained (or renewed). This is useful if you need to restart a service using the certificate.

> For more options, visit the man page for *getcert-request*.
{.is-info}

Verify the certificate.

```bash
getcert list -i LDAP
```

Test renewal of the certificate.

```bash
getcert resubmit -i LDAP
```
