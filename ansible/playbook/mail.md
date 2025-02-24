---
title: Playbook: Mail services with Postfix and Dovecot
description: Postfix and Dovecot Ansible playbook with LDAP authentication and SQL alias mapping
published: true
date: 2025-02-24T13:30:02.639Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-02-24T13:30:02.639Z
---

# Playbook

```yaml
#
# Setting up MariaDB SQL server for alias handling
#

- name: Setting up SQL
  hosts: LIN-SRV1.LEGO.DK
  tasks:
    - name: Installing MariaDB
      ansible.builtin.apt:
        name:
          - mariadb-server
        state: present

    - name: Creating directory for SQL scripts
      ansible.builtin.file:
        path: /sql
        mode: "755"
        state: directory

    - name: Creating database and user template
      ansible.builtin.template:
        src: templates/sql/db-user.sql.j2
        dest: /sql/db-user.sql
        mode: "644"
      vars:
        database: "{{ mail.sql_db }}"
        username: "{{ mail.sql_user }}"
        password: "{{ mail.sql_pass }}"
        location: "127.0.0.1"
        privilege: SELECT
      notify: db_create

    - name: Create table definition template
      ansible.builtin.template:
        src: templates/sql/table.sql.j2
        dest: /sql/table.sql
        mode: "644"
      vars:
        table: postfix_aliases
        columns:
          - id INT AUTO_INCREMENT PRIMARY KEY
          - alias VARCHAR(255) NOT NULL
          - destination VARCHAR(255) NOT NULL
          - active BOOLEAN DEFAULT 1
          - INDEX (alias)
      notify: table_create

    - name: Create table data template
      ansible.builtin.template:
        src: templates/sql/table-insert.sql.j2
        dest: /sql/table-insert.sql
        mode: "644"
      vars:
        table: postfix_aliases
        columns: alias, destination, active
        values:
          - "'johan.bak@lego.dk', 'johan@lego.dk', 1"
          - "'magnus.holm@lego.dk', 'magnus@lego.dk', 1"
          - "'noah.klausen@lego.dk', 'noah@lego.dk', 1"
          - "'group@lego.dk', 'noah@lego.dk', 1"
          - "'group@lego.dk', 'johan@lego.dk', 1"
          - "'group@lego.dk', 'magnus@lego.dk', 1"
      notify: table_fill

  handlers:
    - name: Creating database and user
      listen: db_create
      ansible.builtin.shell:
        cmd: mysql -u root < /sql/db-user.sql
      changed_when: true

    - name: Creating table
      listen: table_create
      ansible.builtin.shell:
        cmd: mysql -u root postfix < /sql/table.sql
      changed_when: true

    - name: Filling table with data
      listen: table_fill
      ansible.builtin.shell:
        cmd: mysql -u root postfix < /sql/table-insert.sql
      changed_when: true

#
# Setting up Postfix for handling incoming mail on SRV1
#

- name: Setting up Postfix
  hosts: LIN-SRV1.LEGO.DK
  tasks:
    - name: Installing Postfix
      ansible.builtin.apt:
        name:
          - postfix
          - postfix-ldap
          - postfix-mysql

    - name: Configuring SMTPD sender login maps
      ansible.builtin.template:
        src: templates/mail/postfix_sender_login_maps.j2
        dest: /etc/postfix/smtpd_sender_login_maps.conf
        mode: "644"
      notify: Restarting Postfix

    - name: Configuring Virtual Alias maps
      ansible.builtin.template:
        src: templates/mail/postfix_virtual_alias_maps.j2
        dest: /etc/postfix/virtual_alias_maps.conf
        mode: "644"
      notify: Restarting Postfix

    - name: Configuring Postfix (main.cf)
      ansible.builtin.template:
        src: templates/mail/postfix_main.cf.j2
        dest: /etc/postfix/main.cf
        mode: "644"
      notify: Restarting Postfix

    - name: Configuring Postfix (master.cf)
      ansible.builtin.copy:
        src: configs/postfix_master.cf
        dest: /etc/postfix/master.cf
        mode: "644"
      notify: Restarting Postfix

  handlers:
    - name: Restarting Postfix
      ansible.builtin.service:
        name: postfix
        state: restarted

#
# Setting up Dovecot on srv2 to handle IMAP and LDAP authentication
#

- name: Setting up Dovecot
  hosts: LIN-SRV2.LEGO.DK
  tasks:
    - name: Installing packages
      ansible.builtin.apt:
        name:
          - dovecot-core
          - dovecot-imapd
          - dovecot-ldap
          - dovecot-lmtpd
        state: present

    - name: Configuring Dovecot (10-mail.conf)
      ansible.builtin.copy:
        src: configs/dovecot_10-mail.conf
        dest: /etc/dovecot/conf.d/10-mail.conf
        mode: "644"
      notify: Restarting Dovecot

    - name: Creating mailbox directory
      ansible.builtin.file:
        path: /mailboxes
        mode: "777"
        state: directory

    - name: Configuring Dovecot (10-ssl.conf)
      ansible.builtin.copy:
        src: configs/dovecot_10-ssl.conf
        dest: /etc/dovecot/conf.d/10-ssl.conf
        mode: "644"
      notify: Restarting Dovecot

    - name: Configuring Dovecot (10-auth.conf)
      ansible.builtin.copy:
        src: configs/dovecot_10-auth.conf
        dest: /etc/dovecot/conf.d/10-auth.conf
        mode: "644"
      notify: Restarting Dovecot

    - name: Configuring Dovecot (dovecot-ldap.conf.ext)
      ansible.builtin.template:
        src: templates/mail/dovecot_ldap.conf.ext.j2
        dest: /etc/dovecot/dovecot-ldap.conf.ext
        mode: "644"
      notify: Restarting Dovecot

    - name: Configuring Dovecot (10-master.conf)
      ansible.builtin.copy:
        src: configs/dovecot_10-master.conf
        dest: /etc/dovecot/conf.d/10-master.conf
        mode: "644"
      notify: Restarting Dovecot

    - name: Configuring Dovecot (dovecot.conf)
      ansible.builtin.copy:
        src: configs/dovecot.conf
        dest: /etc/dovecot/dovecot.conf
        mode: "644"
      notify: Restarting Dovecot

  handlers:
    - name: Restarting Dovecot
      ansible.builtin.service:
        name: dovecot
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

<kbd>templates/sql/db-user.sql.j2</kbd>

```sql
CREATE DATABASE {{ database }};
CREATE USER '{{ username }}'@'{{ location }}' IDENTIFIED BY '{{ password }}';
GRANT {{ privilege }} ON {{ database }}.* TO '{{ username }}'@'{{ location }}';
FLUSH PRIVILEGES;
```

<kbd>templates/sql/table.sql.j2</kbd>

```sql
CREATE TABLE {{ table }} (
{% for row in columns %}
    {{ row }}{% if not loop.last %},{% endif %}
{% endfor %}
);
```

<kbd>templates/sql/table-insert.sql.j2</kbd>

```sql
INSERT INTO {{ table }} ({{ columns }}) VALUES
{% for val in values %}
    ({{val}}){% if not loop.last %},{% endif %}{% if loop.last %};{% endif %}
{% endfor %}
```

<kbd>templates/mail/postfix_sender_login_maps.j2</kbd>

```ini
server_host = ldaps://lin-srv1.lego.dk
version = 3

bind = yes
bind_dn = cn=admin,{{ ldap.base }}
bind_pw = Passw0rd

search_base = ou=Users,{{ ldap.base }}
scope = sub

query_filter = (|(mail=%s)(uid=%s))
result_attribute = mail

```

<kbd>templates/mail/postfix_virtual_alias_maps.j2</kbd>

```ini
user = {{ mail.sql_user }}
password = {{ mail.sql_pass }}
hosts = 127.0.0.1
dbname = {{ mail.sql_db }}
query = SELECT destination FROM postfix_aliases WHERE alias='%s' AND active=1
```

<kbd>templates/mail/postfix_main.cf.j2</kbd>

```ini
smtpd_banner = $myhostname ESMTP $mail_name (Debian/GNU)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 3.6 on
# fresh installs.
compatibility_level = 3.6

# TLS parameters
smtpd_tls_cert_file=/cert/serverCert.pem
smtpd_tls_key_file=/cert/serverKey.pem
smtpd_tls_security_level=encrypt

smtp_tls_CApath=/etc/ssl/certs
smtp_tls_security_level=may
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
mydestination = localhost, localhost.localdomain
relayhost =
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 10.0.0.0/8 [2001:db8:1001::]/48
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all

myorigin = {{ mail.domain }}
mydomain = {{ mail.domain }}
myhostname = {{ mail.smtp_host }}

virtual_mailbox_domains = $mydomain

# Use this line to transfer all mails to Dovecot
virtual_transport = lmtp:inet:{{ mail.imap_host }}

virtual_alias_maps = mysql:/etc/postfix/virtual_alias_maps.conf
smtpd_sender_login_maps = ldap:/etc/postfix/smtpd_sender_login_maps.conf

smtpd_sasl_type = dovecot
smtpd_sasl_path = inet:{{ mail.imap_host }}:12345
smtpd_sasl_auth_enable = yes
broken_sasl_auth_clients = yes

# Logging is optional but can help A LOT with debugging
maillog_file = /var/log/postfix.log
```

<kbd>configs/postfix_master.cf</kbd>

```bash
smtp      inet  n       -       y       -       -       smtpd
submission inet n       -       y       -       -       smtpd
  -o smtpd_tls_security_level=encrypt
  -o smtpd_tls_auth_only=yes
submissions     inet  n       -       y       -       -       smtpd
  -o smtpd_tls_wrappermode=yes
pickup    unix  n       -       y       60      1       pickup
cleanup   unix  n       -       y       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
tlsmgr    unix  -       -       y       1000?   1       tlsmgr
rewrite   unix  -       -       y       -       -       trivial-rewrite
bounce    unix  -       -       y       -       0       bounce
defer     unix  -       -       y       -       0       bounce
trace     unix  -       -       y       -       0       bounce
verify    unix  -       -       y       -       1       verify
flush     unix  n       -       y       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       y       -       -       smtp
relay     unix  -       -       y       -       -       smtp
        -o syslog_name=postfix/$service_name
showq     unix  n       -       y       -       -       showq
error     unix  -       -       y       -       -       error
retry     unix  -       -       y       -       -       error
discard   unix  -       -       y       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       y       -       -       lmtp
anvil     unix  -       -       y       -       1       anvil
scache    unix  -       -       y       -       1       scache
postlog   unix-dgram n  -       n       -       1       postlogd
maildrop  unix  -       n       n       -       -       pipe
  flags=DRXhu user=vmail argv=/usr/bin/maildrop -d ${recipient}
uucp      unix  -       n       n       -       -       pipe
  flags=Fqhu user=uucp argv=uux -r -n -z -a$sender - $nexthop!rmail ($recipient)
#
# Other external delivery methods.
#
ifmail    unix  -       n       n       -       -       pipe
  flags=F user=ftn argv=/usr/lib/ifmail/ifmail -r $nexthop ($recipient)
bsmtp     unix  -       n       n       -       -       pipe
  flags=Fq. user=bsmtp argv=/usr/lib/bsmtp/bsmtp -t$nexthop -f$sender $recipient
scalemail-backend unix -       n       n       -       2       pipe
  flags=R user=scalemail argv=/usr/lib/scalemail/bin/scalemail-store ${nexthop} ${user} ${extension}
mailman   unix  -       n       n       -       -       pipe
  flags=FRX user=list argv=/usr/lib/mailman/bin/postfix-to-mailman.py ${nexthop} ${user}
```


<kbd>configs/dovecot_10-mail.conf</kbd>

```c
mail_location = maildir:/mailboxes/%u

namespace inbox {
  inbox = yes
}

mail_privileged_group = mail

protocol !indexer-worker {

}
```

<kbd>configs/dovecot_10-ssl.conf</kbd>

```c
ssl = yes

ssl_cert = </cert/serverCert.pem
ssl_key = </cert/serverKey.pem

ssl_client_ca_dir = /etc/ssl/certs
ssl_dh = </usr/share/dovecot/dh.pem
```

<kbd>configs/dovecot_10-auth.conf</kbd>

```c
disable_plaintext_auth = no
auth_mechanisms = plain login

!include auth-ldap.conf.ext
```

<kbd>templates/mail/dovecot_ldap.conf.ext.j2</kbd>

```ini
uris = ldaps://lin-srv1.lego.dk ldaps://lin-srv2.lego.dk
dn = cn=admin,{{ ldap.base }}
dnpass = {{ ldap.passwordClear }}

auth_bind = yes
auth_bind_userdn = uid=%n,ou=Users,{{ ldap.base }}

base = ou=Users,{{ ldap.base }}
scope = subtree

user_attrs = homeDirectory=home,uidNumber=uid,gidNumber=gid
user_filter = (&(objectClass=posixAccount)(mail=%u))

pass_attrs = mail=user,userPassword=password
pass_filter = (&(objectClass=posixAccount)(mail=%u))

```

<kbd>configs/dovecot_10-master.conf</kbd>

```c
service imap-login {
  inet_listener imap {
  }
  inet_listener imaps {
  }
}
service pop3-login {
  inet_listener pop3 {
  }
  inet_listener pop3s {
  }
}
service submission-login {
  inet_listener submission {
  }
}
service lmtp {
  unix_listener lmtp {
  }
  inet_listener lmtp {
    port = 24
  }
}
service imap {
}
service pop3 {
}
service submission {
}
service auth {
  unix_listener auth-userdb {
  }
  inet_listener {
    port = 12345
  }
}
service auth-worker {
}
service dict {
  unix_listener dict {
  }
}
```

<kbd>configs/dovecot.conf</kbd>

```ini
!include_try /usr/share/dovecot/protocols.d/*.protocol

dict {
  #quota = mysql:/etc/dovecot/dovecot-dict-sql.conf.ext
}

!include conf.d/*.conf

!include_try local.conf

protocols = imap lmtp

```

