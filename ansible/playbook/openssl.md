---
title: Playbook: OpenSSL Certificate creation
description: Ansible playbook for creating certificates with OpenSSL modules
published: true
date: 2025-02-21T06:58:11.905Z
tags: linux, ansible
editor: markdown
dateCreated: 2025-02-21T06:58:11.905Z
---

> This guide assumes you already have a self-signed CA certificate, which will be used to sign requests.
{.is-info}

# Playbook

```yaml
---
- name: Creating certificates
  hosts: lin_servers
  tasks:
    - name: Generating private keys
      delegate_to: localhost
      community.crypto.openssl_privatekey:
        path: /home/administrator/ansible/certs/{{ inventory_hostname }}.key

    - name: Creating certificate signing requests
      delegate_to: localhost
      community.crypto.openssl_csr:
        path: /home/administrator/ansible/tmp/{{ inventory_hostname }}.csr
        privatekey_path: /home/administrator/ansible/certs/{{ inventory_hostname }}.key
        basic_constraints:
          - CA:FALSE
        key_usage:
          - keyEncipherment
          - dataEncipherment
          - digitalSignature
          - nonRepudiation
        extended_key_usage:
          - clientAuth
          - serverAuth
        subject: "{{ subject }}"
        subject_alt_name: "{{ subjectAltName }}"

    - name: Signing certificates
      delegate_to: localhost
      community.crypto.x509_certificate:
        path: /home/administrator/ansible/certs/{{ inventory_hostname }}.crt
        csr_path: /home/administrator/ansible/tmp/{{ inventory_hostname }}.csr
        ownca_path: /home/administrator/ansible/certs/subca/subca.crt
        ownca_privatekey_path: /home/administrator/ansible/certs/subca/subca.key
        provider: ownca

    - name: Reading certificates
      delegate_to: localhost
      changed_when: false
      register: certchain
      ansible.builtin.command: >
        cat
        /home/administrator/ansible/certs/{{ inventory_hostname }}.crt
        /home/administrator/ansible/certs/subca/subca.crt
        /home/administrator/ansible/certs/ca.pem

    - name: Creating certificate chains
      delegate_to: localhost
      ansible.builtin.copy:
        dest: /home/administrator/ansible/certs/{{ inventory_hostname }}_chain.crt
        content: "{{ certchain.stdout }}"
        mode: "644"

    - name: Creating target directory
      ansible.builtin.file:
        path: /cert
        mode: "755"
        state: directory

    - name: Distributing CA certificate [/cert]
      ansible.builtin.copy:
        src: certs/ca.pem
        dest: /cert/ca.pem
        mode: "644"

    - name: Distributing CA certificate [/usr/local/share/ca-certificates]
      ansible.builtin.copy:
        src: certs/ca.pem
        dest: /usr/local/share/ca-certificates/ca.crt
        mode: "644"
      notify: Import CA

    - name: Distributing SubCA certificate [/cert]
      ansible.builtin.copy:
        src: certs/subca/subca.crt
        dest: /cert/subca.pem
        mode: "644"

    - name: Distributing SubCA certificate [/usr/local/share/ca-certificates]
      ansible.builtin.copy:
        src: certs/subca/subca.crt
        dest: /usr/local/share/ca-certificates/subca.crt
        mode: "644"
      notify: Import CA

    - name: Distributing certificates
      ansible.builtin.copy:
        src: certs/{{ inventory_hostname }}_chain.crt
        dest: /cert/serverCert.pem
        mode: "644"

    - name: Distributing keys
      ansible.builtin.copy:
        src: certs/{{ inventory_hostname }}.key
        dest: /cert/serverKey.pem
        mode: "600"

  handlers:
    - name: Import CA
      ansible.builtin.command: update-ca-certificates
      changed_when: true
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
      subject:
        C: DK
        O: LEGO
        CN: lin-srv1.lego.dk
      subjectAltName:
        - DNS:lin-srv1.lego.dk
        - IP:10.0.0.101
    LIN-SRV2.LEGO.DK:
      ansible_host: 10.10.0.102
      address: 10.10.0.102
      hostname: lin-srv2
      subject:
        C: DK
        O: LEGO
        CN: lin-srv2.lego.dk
      subjectAltName:
        - DNS:lin-srv2.lego.dk
        - IP:10.0.0.102
```