---
title: Windows PKI
description: Two-tiered AD CS PKI with CDP and AIA
published: true
date: 2025-06-20T08:42:47.851Z
tags: windows
editor: markdown
dateCreated: 2025-04-22T20:06:32.021Z
---

# Information

This guide will demonstrate how to create a two-tiered **Windows AD CS** PKI with **CDP** and **AIA** extensions set up correctly. The root certification authority will be an "offline" CA.

Two Windows Server 2022 VMs will be used:

- **srv-root** - `10.0.0.1/24`
- **srv-signing** - `10.0.0.2/24`

# Prerequisites

As with any PKI setup, always make sure that the current time is correct on all machines.

For the enterprise issuing CA, you will need **AD DS** installed, in this case, on **srv-signing**. This guide will use the domain **leg<span>o.d</span>k**. The server was promoted to be the **DC** of this forest.

For name resolution, **srv-signing** is set as the DNS server for **srv-root**.

This setup will use the **FQDN** `pki.lego.dk` for distributing certificates. Create an *A/CNAME* record for it that points to **srv-signing**.

# Setup

## IIS

First, we need to set up a **web server** to serve as a distribution point. On **srv-signing**, **install IIS with default settings**.

**Create** the folder `C:\pki`. **Right click** on it, and click on ***Give access to people...* > *Specific People***. **Add** the following members:

- **Cert Publishers** - Read/Write
- **ANONYMOUS LOGON** - Read
- **Everyone** - Read

**Open** the IIS management console. Under the **default website**, create a new **virtual directory** with the following parameters:

- **Alias:** `pki`
- **Physical path:** `C:\pki`

In the virtual directory menu, double click on **Request Filtering**. On the right side, **edit feature settings**. In the pop-up window, make sure **Allow double escaping** is checked.

In the IIS tree view on the left, click **SRV-SIGNING** *(your server name)* and on the right, click **Restart**.

IIS is now ready to serve requests.

## Root certification authority

On **srv-root**, install **AD CS** with all default settings. Only install the **Certification Authority** role feature.

When setting up the authority, select **Standalone CA** *(you cannot select Enterprise CA as this machine is not domain-joined)* and **Root CA** options.

When entering the **name** of the CA, for convenience, **DO NOT use any whitespace**. The guide will use `lego-rca`.

After the setup is complete, open the **Certification Authority** management console, then right-click on the CA and select **Properties**. Open the **Extensions** pane.

**With the CDP extension selected:**

- **Delete** all locations **except the first one** *(starting with `C:\...`)*
- **Add a new location:** `http://pki.lego.dk/pki/<CaName><CRLNameSuffix><DeltaCRLAllowed>.crl` *(For the variable part, just follow the example)*
  - With this location selected, **check**:
    - *Include in CRLs. Clients use this to find Delta CRL locations.*
    - *Include in the CDP extension of issued certificates.*

> **CDP** and **AIA** extensions only work over **HTTP** and **not HTTPS**!
{.is-warning}

**With the AIA extension selected:**

- **Delete** all locations **except the first one** *(starting with `C:\...`)*
- **Add a new location:** `http://pki.lego.dk/pki/lego-rca<CertificateName>.crt`
  - With this location selected, **check**:
    - *Include in the AIA extension of issued certificates.*

Click **OK** and **restart** the CA when prompted.

In a terminal with admin privileges, issue the following command:

```ps1
certutil -crl
```

Navigate to `C:\Windows\System32\Certsrv\CertEnroll\` in File Explorer. There should be two files, with **.crt** and **.crl** file extensions. Copy both files to the `pki` share on **srv-signing**. Rename the root certificate file to `lego-rca.crt`.

> The root certificate filename should match the filename you wrote in the AIA extension.
>
> In that location, the `<CertificateName>` variable only contains text after the root CA is renewed, right now it is an empty string.
{.is-info}

## Issuing certification authority

After the root certification authority is set up, you can move on to configuring the issuing (subordinate) certification authority. The first task is to **import the root certificate** to the **Trusted Root Certification Authorities** store. The file is already present at `C:\pki\lego-rca.crt`. Right click and install it. *(Local Machine, Trusted Root Certification Authorities)*

> The root certificate needs to be trusted otherwise importing the signed subordinate certification authority certificate will fail.
{.is-info}

Now **install AD CS** role on the server, only selecting the **Certification Authority** role feature.

**When configuring the role, select:**

- Enterprise CA
- Subordinate CA
- For convenience, select a name **WITHOUT whitespace characters**. This guide will use `lego-signing-ca`

**Export the signing request** into a file. Copy that file over to **srv-root** and in its CA console, **submit the request**. After submission, **issue** the certificate from the pending certificates. After issuing, **open the certificate** *(in Issued Certificates)* and **export it** into a file. *(Details pane > Copy to File...)*

**Copy** the signed certificate back to **srv-signing**, then in the CA console, **install the certificate**. If everything so far was done correctly, there should be no warnings given, because the certificate engine should be able to reach the CDP of the root cert to check the validity of the subordinate certificate.

After the Certification Authority service has started, right click on the CA and open **Properties**. Open the **Extensions** pane.

**With the CDP extension selected:**

- **Delete** all locations **except the first one** *(starting with `C:\...`)*
- **Add a new location:** `http://pki.lego.dk/pki/<CaName><CRLNameSuffix><DeltaCRLAllowed>.crl` *(For the variable part, just follow the example)*
  - With this location selected, **check**:
    - *Include in CRLs. Clients use this to find Delta CRL locations.*
    - *Include in the CDP extension of issued certificates.*
- **Add a new location:** `C:\pki\<CaName><CRLNameSuffix><DeltaCRLAllowed>.crl` *(The filename part is the same)*
  - With this location selected, **check**:
    - *Publish CRLs to this location.*
    - *Publish Delta CRLs to this location.*

> To publish the CRL to a **remote location**, you can use the `file:\\server-name\share-name\...` format after creating the share and granting write access for the **Cert Publishers** group.
{.is-info}

**With the AIA extension selected:**

- **Delete** all locations **except the first one** *(starting with `C:\...`)*
- **Add a new location:** `http://pki.lego.dk/pki/lego-signing-ca<CertificateName>.crt`
  - With this location selected, **check**:
    - *Include in the AIA extension of issued certificates.*

Click **OK** and **restart** the CA when prompted.

In a terminal with admin privileges, issue the following command:

```ps1
certutil -crl
```

Navigate to `C:\Windows\System32\Certsrv\CertEnroll\` in File Explorer. There should be two files, with **.crt** and **.crl** file extensions. Copy both files to `C:\pki\`. *(Although the .crl file should already be present, after running the certutil command.)* **Rename** the certificate file to `lego-signing-ca.crt`.

Now, you have **CDP** and **AIA** extensions configured, clients will be able to check revocation and obtain the certificate chain to verify trust.

# Verification

To verify your setup, open `pkiview.msc`. After loading, all of the **Status** fields should be **OK** when navigating through the trust chain.
