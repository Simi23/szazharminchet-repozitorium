---
title: Playbook: Roundcube Webmail
description: Roundcube Ansible playbook
published: true
date: 2025-02-24T13:36:48.104Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-02-24T13:36:48.104Z
---

# Playbook

```yaml
#
# Installing Roundcube for webmail
#

- name: Installing Roundcube
  hosts: LIN-SRV1.LEGO.DK
  tasks:
    - name: Installing packages
      ansible.builtin.apt:
        name: roundcube
        state: present

    - name: Configuring Roundcube
      ansible.builtin.template:
        src: templates/mail/roundcube.php.j2
        dest: /etc/roundcube/config.inc.php
        mode: "644"
      notify: Restarting Apache2

    - name: Configuring Apache2 VirtualHost
      ansible.builtin.template:
        src: templates/mail/roundcube-vhost.conf.j2
        dest: /etc/apache2/sites-available/roundcube.conf
        mode: "644"
      notify: Restarting Apache2

    - name: Enabling SSL mod
      community.general.apache2_module:
        name: ssl
        state: present
      notify: Restarting Apache2

    - name: Deleting default virtual hosts
      ansible.builtin.file:
        path: /etc/apache2/sites-enabled/000-default.conf
        state: absent
      notify: Restarting Apache2

    - name: Linking Roundcube VirtualHost
      ansible.builtin.file:
        path: /etc/apache2/sites-enabled/roundcube.conf
        src: /etc/apache2/sites-available/roundcube.conf
        state: link
        mode: "644"
      notify: Restarting Apache2

  handlers:
    - name: Restarting Apache2
      ansible.builtin.service:
        name: apache2
        state: restarted

```

# Inventory

```yaml
lin_servers:
  vars:
    ansible_user: root
    ansible_password: Passw0rd
    ansible_python_interpreter: auto_silent
    domain: lego.dk
  hosts:
    LIN-SRV1.LEGO.DK:
      ansible_host: 10.10.0.101
      address: 10.10.0.101
      hostname: lin-srv1
      dns_role: master
      ldap_role: provider
      subject:
        C: DK
        O: LEGO
        CN: lin-srv1.lego.dk
      subjectAltName:
        - DNS:lin-srv1.lego.dk
        - DNS:smtp.lego.dk
        - DNS:mail.lego.dk
        - IP:10.0.0.101
    LIN-SRV2.LEGO.DK:
      ansible_host: 10.10.0.102
      address: 10.10.0.102
      hostname: lin-srv2
      dns_role: slave
      ldap_role: consumer
      subject:
        C: DK
        O: LEGO
        CN: lin-srv2.lego.dk
      subjectAltName:
        - DNS:lin-srv2.lego.dk
        - DNS:imap.lego.dk
        - IP:10.0.0.102

all:
  vars:
    dns:
      allow_forward: true
      forwarders:
        - 20.0.0.1
      masters:
        - 10.10.0.101
      slaves:
        - 10.10.0.102
      zones:
        - name: "lego.dk"
          file: "/etc/bind/db.lego.dk"
          records:
            - "@          NS  lin-srv1.lego.dk."
            - "@          NS  lin-srv2.lego.dk."
            - "@          MX  10 smtp.lego.dk."
            - "lin-srv1   A   10.10.0.101"
            - "smtp       A   10.10.0.101"
            - "lin-srv2   A   10.10.0.102"
            - "imap       A   10.10.0.102"
            - "lin-clt    A   10.10.0.1"
            - "rtr        A   10.10.0.254"
            - "mail       CNAME lin-srv1.lego.dk."
            - "$ORIGIN _tcp.lego.dk."
            - "_imap        SRV  10  0  143  imap.lego.dk."
            - "_imaps       SRV  10  0  993  imap.lego.dk."
            - "_submission  SRV  10  0  587  smtp.lego.dk."
            - "_smtps       SRV  10  0  465  smtp.lego.dk."

        - name: "0.10.10.in-addr.arpa"
          file: "/etc/bind/db.10.10.0"
          records:
            - "@    NS  lin-srv1.lego.dk."
            - "@    NS  lin-srv2.lego.dk."
            - "101  PTR lin-srv1.lego.dk."
            - "102  PTR lin-srv2.lego.dk."
            - "1    PTR lin-clt.lego.dk."
            - "254  PTR rtr.lego.dk."
    ldap:
      password: "{SSHA}DlLGOla4ikFoP1wfPXAySfsj2rBrWaZ+" # Passw0rd
      passwordClear: Passw0rd
      master: lin-srv1.lego.dk
      base: dc=lego,dc=dk
      # password: "{SSHA}4F5ESLULjeyEXBg8C3r+giOpyESfyQon" # 12345678
      users:
        - uid: noah
          uid_number: 10001
          gid_number: 5001
          first_name: Noah
          last_name: Klausen
          mail: noah@lego.dk
        - uid: johan
          uid_number: 10002
          gid_number: 5001
          first_name: Johan
          last_name: Bak
          mail: johan@lego.dk
        - uid: magnus
          uid_number: 10003
          gid_number: 5001
          first_name: Magnus
          last_name: Holm
          mail: magnus@lego.dk
    
    mail:
      sql_user: postfix_user
      sql_pass: Passw0rd
      sql_db: postfix
      domain: lego.dk
      smtp_host: smtp.lego.dk
      imap_host: imap.lego.dk
      webmail_host: mail.lego.dk

```

# Templates

<kbd>templates/mail/roundcube.php.j2</kbd>

```php
<?php

$config = [];

include("/etc/roundcube/debian-db-roundcube.php");
$config['imap_host'] = ["tls://{{ mail.imap_host }}"];
$config['smtp_host'] = 'tls://{{ mail.smtp_host }}';
$config['smtp_user'] = '%u';
$config['smtp_pass'] = '%p';
$config['support_url'] = '';
$config['product_name'] = 'LEGO.DK Webmail';
$config['des_key'] = 'm7La1EMqm/xE8fBbTbd+4PIt';
$config['plugins'] = [
    'archive',
    'zipdownload',
];
$config['skin'] = 'elastic';
$config['enable_spellcheck'] = false;
$config['language'] = 'en_US';

```

<kbd>templates/mail/roundcube-vhost.conf.j2</kbd>

```ini
<VirtualHost *:80>
	ServerName {{ mail.webmail_host }}
	Redirect / https://{{ mail.webmail_host }}/
</VirtualHost>

<VirtualHost *:443>
	ServerAdmin webmaster@localhost

	ServerName {{ mail.webmail_host }}
	DocumentRoot /var/lib/roundcube

	ErrorLog ${APACHE_LOG_DIR}/rc_error.log
	CustomLog ${APACHE_LOG_DIR}/rc_access.log combined

	<Directory /var/lib/roundcube>
		Options FollowSymLinks
		AllowOverride All
		Require all granted
	</Directory>

	SSLEngine on

	SSLCertificateFile      /cert/serverCert.pem
	SSLCertificateKeyFile   /cert/serverKey.pem

	SSLCertificateChainFile /cert/serverCert.pem

	<FilesMatch "\.(?:cgi|shtml|phtml|php)$">
		SSLOptions +StdEnvVars
	</FilesMatch>
	<Directory /usr/lib/cgi-bin>
		SSLOptions +StdEnvVars
	</Directory>
  
</VirtualHost>
```
