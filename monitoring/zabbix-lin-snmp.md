---
title: Linux SNMP monitoring
description: How to set up Linux SNMP monitoring.
published: true
date: 2025-06-11T13:52:21.796Z
tags: linux
editor: markdown
dateCreated: 2025-06-11T13:38:43.399Z
---

# Zabbix SNMP monitoring (LINUX)

## Zabbix

> Under `Configuration\Hosts` tab, you can create new hosts. There are some variables you have to set, and after that you can use the default templates to monitor through SNMP or, you can set up new SNMP OID's to monitor.
{.is-info}

- **Host name**: Set up the name it will be the display name and some variables will use this as reference.
- **Templates**: Choose the templates you have created, that has the items and triggers you will want for this host.
- **Groups**: Choose a group to use for this host.
- **Interfaces**: Add a new interface, choose SNMP. Set **IP** address or **DNS** name, and CHOOSE which one you want to use. Set up the port of the server (161 for SNMP). Choose SNMP version, and fill the fields with your datas. Here is one for SNMPv3

| Option name | Value |
| :---------: | :---: |
| Security level | authPriv |
| Context name | *empty* |
| Security name | *Username* |
| Authentication protocol | *MD5* |
| Authentication password | *User auth password* |
| Privay protocol | *AES256C* |
| Privay password | *User privacy password* |

## SNMPD
> Install `snmp`, and `snmpd` packages, edit the `/etc/snmp/snmpd.conf` file.
{.is-info}

```conf
sysLocation YOURSYSTEMLOCATION
sysContact NAME <email@address>

agentaddress 0.0.0.0, [::]

createuser Administrator MD5 "Passw0rd!" AES256C "Passw0rd!"
rouser Administrator authpriv
```

> Restart snmpd service.
{.is-info}

```bash
systemctl restart snmpd
```

