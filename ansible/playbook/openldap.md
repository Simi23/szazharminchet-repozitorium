---
title: Playbook: OpenLDAP w/ TLS & Replication
description: Ansible playbook for configuring OpenLDAP on two machines with TLS and replication
published: true
date: 2025-02-21T07:09:20.945Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-02-21T07:09:20.945Z
---

# Playbook

```yaml
- name: Installing OpenLDAP
  hosts: lin_servers
  become: true
  tasks:
    - name: Installing packages
      ansible.builtin.apt:
        name:
          - slapd
          - ldap-utils
          - python3-ldap

    - name: Setting root password
      community.general.ldap_attrs:
        dn: olcDatabase={1}mdb,cn=config
        attributes:
          olcRootPW: "{{ ldap.password }}"
        state: exact

    - name: Copying keyfile
      ansible.builtin.copy:
        src: /cert/serverKey.pem
        remote_src: true
        dest: /cert/ldapKey.pem
        mode: "600"
        owner: openldap
        group: openldap

    - name: Setting TLS Certificate information
      community.general.ldap_attrs:
        dn: cn=config
        attributes:
          olcTLSCACertificateFile: /cert/ca.pem
          olcTLSCertificateKeyFile: /cert/ldapKey.pem
          olcTLSCertificateFile: /cert/serverCert.pem
        state: exact
      notify: slapd_restart

    - name: Enabling ldaps:/// schema
      ansible.builtin.copy:
        src: configs/slapd.default
        dest: /etc/default/slapd
        mode: "644"
      notify: slapd_restart

    - name: Reading LDAP Roles
      ansible.builtin.group_by:
        key: "ldap_server_{{ ldap_role }}"
      changed_when: false

  handlers:
    - name: Restarting OpenLDAP Daemon
      ansible.builtin.service:
        name: slapd
        state: restarted
      listen: slapd_restart

- name: Setting up LDAP Sync (1/2 - Provider)
  hosts: ldap_server_provider
  tasks:
    - name: Creating replication user
      community.general.ldap_entry:
        bind_dn: cn=admin,{{ ldap.base }}
        bind_pw: "{{ ldap.passwordClear }}"
        objectClass:
          - simpleSecurityObject
          - organizationalRole
        dn: cn=replicator,{{ ldap.base }}
        attributes:
          cn: replicator
          description: Replication user
          userPassword: "{{ ldap.password }}"

    - name: Creating ACLs and indexes
      community.general.ldap_attrs:
        dn: olcDatabase={1}mdb,cn=config
        attributes:
          olcAccess: >-
            {0}to *
            by dn.exact="cn=replicator,{{ ldap.base }}" read
            by * break
          olcLimits: >-
            dn.exact="cn=replicator,{{ ldap.base }}"
            time.soft=unlimited time.hard=unlimited
            size.soft=unlimited size.hard=unlimited
          olcDbIndex:
            - entryCSN eq
            - entryUUID eq
        state: present

    - name: Loading syncprov module
      community.general.ldap_attrs:
        dn: cn=module{0},cn=config
        attributes:
          olcModuleLoad: syncprov

    - name: Configuring syncprov module
      community.general.ldap_entry:
        dn: olcOverlay=syncprov,olcDatabase={1}mdb,cn=config
        objectClass:
          - olcOverlayConfig
          - olcSyncProvConfig
        attributes:
          olcOverlay: syncprov
          olcSpCheckpoint: "100 10"
          olcSpSessionLog: "100"

- name: Setting up LDAP Sync (1/2 - Consumer)
  hosts: ldap_server_consumer
  tasks:
    - name: Loading syncprov module
      community.general.ldap_attrs:
        dn: cn=module{0},cn=config
        attributes:
          olcModuleLoad: syncprov

    - name: Updating sync configuration
      community.general.ldap_attrs:
        dn: olcDatabase={1}mdb,cn=config
        attributes:
          olcDbIndex: entryUUID eq
          olcSyncrepl: >-
            rid=0
            provider=ldap://{{ ldap.master }}
            bindmethod=simple
            binddn="cn=replicator,{{ ldap.base }}" credentials={{ ldap.passwordClear }}
            searchbase="{{ ldap.base }}"
            schemachecking=on
            type=refreshAndPersist retry="60 +"
            starttls=critical tls_reqcert=demand
          olcUpdateRef: ldap://{{ ldap.master }}

- name: Populating LDAP database
  hosts: ldap_server_provider
  tasks:
    - name: Creating User OU
      community.general.ldap_entry:
        bind_dn: cn=admin,{{ ldap.base }}
        bind_pw: "{{ ldap.passwordClear }}"
        dn: ou=Users,dc=lego,dc=dk
        objectClass:
          - organizationalUnit
        attributes:
          ou: Users

    - name: Creating Group OU
      community.general.ldap_entry:
        bind_dn: cn=admin,{{ ldap.base }}
        bind_pw: "{{ ldap.passwordClear }}"
        dn: ou=Groups,dc=lego,dc=dk
        objectClass:
          - organizationalUnit
        attributes:
          ou: Groups

    - name: Creating factory group
      community.general.ldap_entry:
        bind_dn: cn=admin,{{ ldap.base }}
        bind_pw: "{{ ldap.passwordClear }}"
        dn: cn=factory,ou=Groups,dc=lego,dc=dk
        objectClass:
          - posixGroup
        attributes:
          cn: factory
          gidNumber: "5001"
          memberUid:
            - noah
            - johan
            - magnus

    - name: Creating Users
      community.general.ldap_entry:
        bind_dn: "cn=admin,{{ ldap.base }}"
        bind_pw: "{{ ldap.passwordClear }}"
        dn: "uid={{ item.uid }},ou=Users,{{ ldap.base }}"
        objectClass:
          - inetOrgPerson
          - posixAccount
          - shadowAccount
        attributes:
          uid: "{{ item.uid }}"
          givenName: "{{ item.first_name }}"
          sn: "{{ item.last_name }}"
          cn: "{{ item.first_name }} {{ item.last_name }}"
          gecos: "{{ item.first_name }} {{ item.last_name }}"
          uidNumber: "{{ item.uid_number }}"
          gidNumber: "{{ item.gid_number }}"
          userPassword: "{{ ldap.password }}"
          loginShell: "/bin/bash"
          homeDirectory: "/home/{{ item.uid }}"
          mail: "{{ item.mail }}"
      loop: "{{ ldap.users }}"
      loop_control:
        label: "{{ item.first_name }} {{ item.last_name }}"
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
      ldap_role: provider
    LIN-SRV2.LEGO.DK:
      ansible_host: 10.10.0.102
      address: 10.10.0.102
      hostname: lin-srv2
      ldap_role: consumer

all:
  vars:
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
```

# Templates

<kbd>configs/slapd.default</kbd>

```c
SLAPD_CONF=
SLAPD_USER="openldap"
SLAPD_GROUP="openldap"
SLAPD_PIDFILE=
SLAPD_SERVICES="ldap:/// ldapi:/// ldaps:///"
SLAPD_SENTINEL_FILE=/etc/ldap/noslapd
SLAPD_OPTIONS=""
```
