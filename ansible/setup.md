---
title: Ansible Setup
description: Ansible environment setup to use with Linux and Windows hosts
published: true
date: 2025-02-20T09:07:55.608Z
tags: linux, windows, ansible
editor: markdown
dateCreated: 2025-02-19T11:59:27.208Z
---

# Basic setup

To create a basic Ansible environment, you need a python **virtual environment**, because you will need to install **pip modules**. First, install **python**.

```bash
apt install python3 python3-pip python3-venv
```

Create a directory, e.g. `ansible`:

```bash
mkdir ansible && cd ansible
```

Create the virtual environment, enter it, then install Ansible. You can choose any name for it, here it is `python-ansible`.

```bash
python3 -m venv python-ansible
source python-ansible/bin/activate
pip install ansible ansible-lint
```

Now you can run your playbooks with `ansible-playbook`. For device specifics, continue reading.

# Cisco devices

To reach Cisco devices, you need a python module. Install it.

```bash
pip install ansible-pylibssh
```

SSH algorithms on Cisco devices are outdated, and deprecated by default OpenSSL settings. You need to enable them in `/etc/ssh/ssh_config`.

```
Host *
    KexAlgorithms +diffie-hellman-group14-sha1
    HostkeyAlgorithms +ssh-rsa
```

Restart SSH service.

```bash
service ssh restart
```

Sample inventory file:

```yaml
cisco:
  vars:
    ansible_connection: ansible.netcommon.network_cli
    ansible_network_os: cisco.ios.ios
    ansible_user: ansible
    ansible_password: ansible
  hosts:
    r1:
      ansible_host: 192.168.86.201
```

> You need enable SSHv2, local authentication and SSH transport on the device. You also need to create a user with privilege 15, the snippet above uses `ansible`/`ansible`.
{.is-warning}

# Linux devices

To reach Linux devices with Ansible with password authentication, you need the `sshpass` package. Install it.

```bash
apt install sshpass
```

Before reaching a new device, you have to put its SSH public key into your `known_hosts` file.

```bash
ssh-keyscan -t rsa rtr.lego.dk >> ~/.ssh/known_hosts
```

> This can also be done with a playbook by delegating this command to localhost.
{.is-info}

Now you can use Ansible to reach the Linux host. Sample inventory:

```yaml
linux:
  vars:
      ansible_user: root
      ansible_password: Passw0rd
      ansible_python_interpreter: auto_silent
  hosts:
    RTR.LEGO.DK:
      ansible_host: rtr.lego.dk
```

# Windows devices

To reach Windows devices **without modifying them first**, you need to use **NTLM** authentication.

Sample inventory file:

```yaml
windows:
  hosts:
    WIN-SRV.TEST.DK:
      ansible_host: WIN-SRV.TEST.DK
      ansible_port: 5985
      ansible_user: Administrator@TEST.DK
      ansible_password: Passw0rd
      ansible_connection: winrm
      ansible_winrm_transport: ntlm
      ansible_become_method: runas
      ansible_become_user: TEST\Administrator
      ansible_become_password: Passw0rd
```

> When using Windows hosts, be sure to use `become: true` in the playbook, as this will prevent permission errors with the WinRM connection by running the commands locally.
{.is-warning}

> When entering the domain/realm name in the inventory, always use **full uppercase** letters!
{.is-warning}
