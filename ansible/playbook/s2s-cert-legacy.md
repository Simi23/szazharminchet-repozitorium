---
title: Playbook: Site-to-site VPN with Cert
description: Site-to-site VPN Ansible playbook for RRAS - Strongswan (legacy)
published: true
date: 2025-02-24T08:36:35.299Z
tags: linux, windows, ansible
editor: markdown
dateCreated: 2025-02-24T08:05:30.200Z
---

# Playbook

```yaml
---
- name: Install Windows CA
  hosts: WIN-SRV-AD.TEST.DK
  gather_facts: false
  become: true
  become_method: runas
  vars:
    cert_requests: "{{ s2s.ids }}"
  tasks:
    - name: Install ADCS
      ansible.windows.win_feature:
        name: Adcs-Cert-Authority
        state: present
        include_management_tools: true

    - name: Check if Root CA installed
      ansible.windows.win_shell: |
        certutil - -ping
      register: cainstalled
      failed_when: false
      changed_when: '"FAILED: 0x80070002 (WIN32: 2 ERROR_FILE_NOT_FOUND)" in cainstalled.stdout'
      notify: Create Root CA

    - name: Run handlers create Root CA
      ansible.builtin.meta: flush_handlers

    - name: Check if IPSec template is already added
      ansible.windows.win_shell: |
        if (!(certutil -catemplates | Select-String -Pattern "IPSECIntermediateOffline")) {
          Write-Host "Disabled"
        }
      register: ipsectemplate
      changed_when: '"Disabled" in ipsectemplate.stdout'
      notify: Enable IPSec template
    
    - name: Enable IPSec template
      ansible.builtin.meta: flush_handlers

    - name: Check if the file exists
      ansible.windows.win_shell: Test-Path C:\lin-s2s.inf
      register: file_info
      changed_when: '"True" not in file_info.stdout'
      notify: Certs


  handlers:
    - name: Create Root CA
      ansible.windows.win_shell: >
        Install-AdcsCertificationAuthority
        -ValidityPeriod Years
        -ValidityPeriodUnits 5
        -CACommonName WIN-SRV-CA
        -CAType EnterpriseRootCA
        -KeyLength 2048
        -Force
      listen: Create Root CA

		- name: Wait for Windows to recognise everything is fine
    	ansible.builtin.pause:
      	seconds: 30
       listen: Enable IPSec template

    - name: Add IPSec to managed templates
      ansible.windows.win_shell: > 
        certutil -setcatemplates +IPSECIntermediateOffline
      listen: Enable IPSec template

    - name: Create multiple certificate request files
      template:
        src: templates/s2s.inf.j2
        dest: C:/{{ item.filename }}
      loop: "{{ cert_requests }}"
      listen: Certs
    
    - name: Create requests for .inf files
      ansible.windows.win_shell: >
        certreq -new C:\{{ item.inf }} C:\{{ item.req }}
      loop: "{{ s2s.files }}"
      listen: Certs
    
    - name: Accept requests
      ansible.windows.win_shell: >
        certreq -submit -config "WIN-SRV-AD.TEST.DK\WIN-SRV-CA" C:\{{ item.req }} C:\{{ item.crt }}
      loop: "{{ s2s.files }}"
      listen: Certs
    
    - name: Add cert to Machine Personal Store
      ansible.windows.win_shell: >
        certreq -accept C:\{{ item.crt }}
      loop: "{{ s2s.files }}"
      listen: Certs

    - name: Export .pfx
      ansible.windows.win_shell: |
        $CertPassword = ConvertTo-SecureString -String "Passw0rd" -Force -AsPlainText
        $Cert = Get-ChildItem -Path Cert:\LocalMachine\My | Where-Object { $_.Subject -match "CN=LIN-RTR.lego.dk" }
        Export-PfxCertificate -Cert $Cert -FilePath "C:\lin-s2s.pfx" -Password $CertPassword
      listen: Certs

    - name: Export Root CA
      ansible.windows.win_shell: |
        $RootCA = (Get-ChildItem -Path Cert:\LocalMachine\Root | Where-Object { $_.Subject -match "CN=WIN-SRV-CA, DC=TEST, DC=DK" })[0]
        Export-Certificate -Cert $RootCA -FilePath "C:\CA.crt"
      listen: Certs

    - name: Fetch files from the Windows server
      fetch:
        src: C:/{{ item.srcname }}
        dest: /root/{{ item.destname }}
        flat: yes
      loop: "{{ s2s.fetchfiles }}"
      listen: Certs


- name: S2S Strongswan Cert - Legacy
  hosts: lin_rtr
  gather_facts: false
  tasks:
    - name: Install Strongswan packages
      ansible.builtin.apt:
        name: 
          - strongswan
          - strongswan-starter
          - libcharon-extra-plugins
        state: present

    - name: Copy ipsec.conf (1/2)
      ansible.builtin.template:
        src: templates/ipsec.conf.s2scert.j2
        dest: /etc/ipsec.conf
        owner: root
        group: root
        mode: "0644"
      notify: 
        - Restart strongswan
    
    - name: Copy ipsec.secrets (2/2)
      ansible.builtin.template:
        src: templates/ipsec.secrets.s2scert.j2
        dest: /etc/ipsec.secrets
        owner: root
        group: root
        mode: "0644"
      notify:
        - Restart strongswan
    
    - name: Copy CA (1/2)
      ansible.builtin.copy:
        src: /root/WIN-CA.crt
        dest: /usr/local/share/ca-certificates/
      notify:
        - Update CA
    
    - name: Copy CA (2/2)
      ansible.builtin.copy:
        src: /root/WIN-CA.crt
        dest: /etc/ipsec.d/cacerts/
      notify:
        - Restart strongswan
       
    - name: Copy pfx file
      ansible.builtin.copy:
        src: /root/lin-s2s.pfx
        dest: /etc/ipsec.d/
      notify: 
        - pkcs12
        - Restart strongswan
 
  handlers:
    - name: Update CA-certificates
      ansible.builtin.shell: update-ca-certificates
      changed_when: false
      listen: Update CA

    - name: Parse private key from PKCS12
      ansible.builtin.command: 
        cmd: openssl pkcs12 -in /etc/ipsec.d/lin-s2s.pfx -out /etc/ipsec.d/private/server.key -nocerts -nodes -passin pass:Passw0rd
        creates: /etc/ipsec.d/private/server.key
      listen: pkcs12
    
    - name: Parse certificate from PKCS12
      ansible.builtin.command:
        cmd: openssl pkcs12 -in /etc/ipsec.d/lin-s2s.pfx -out /etc/ipsec.d/certs/server.crt -nokeys -nodes -passin pass:Passw0rd
        creates: /etc/ipsec.d/certs/server.crt
      listen: pkcs12

    - name: Restart strongswan daemon
      ansible.builtin.service:
        name: strongswan-starter
        state: restarted
      listen: Restart strongswan

- name: Configure RRAS
  hosts: WIN-SRV-AD.TEST.DK
  gather_facts: false
  tasks:
    - name: Install RRAS
      ansible.windows.win_feature:
        name:
          - RemoteAccess
          - Routing
        state: present
        include_management_tools: true

    - name: Check VPN service installation (1/2)
      ansible.windows.win_shell: 'Get-RemoteAccess'
      register: vpn
      changed_when: '"RoutingStatus      : Installed" not in vpn.stdout'
      notify: Configure VPN RoutingOnly
    
    - name: Check VPN service installation (2/2)
      ansible.windows.win_shell: 'Get-RemoteAccess'
      register: vpn
      changed_when: '"VpnS2SStatus       : Installed" not in vpn.stdout'
      notify: Configure VPN S2S
    
    - name: Run handlers for Services
      ansible.builtin.meta: flush_handlers

    - name: Check VPN interface status
      ansible.windows.win_shell: 'Get-VpnS2SInterface | Select-Object -ExpandProperty Name'
      register: vpns2s
      changed_when: '"LIN-RTR.lego.dk" not in vpns2s.stdout'
      notify: Configure S2S Interface
    
    - name: Check VPN protocol status
      ansible.windows.win_shell: 'Get-VpnAuthProtocol | Select-Object -ExpandProperty UserAuthProtocolAccepted'
      register: vpns2scert
      changed_when: '"Certificate" not in vpns2scert.stdout'
      notify: S2S Cert

    - name: Run handlers before start
      ansible.builtin.meta: flush_handlers
    
    - name: Start connection
      win_shell: >
        Connect-VpnS2SInterface -Name "LIN-RTR.lego.dk"
      changed_when: false
    
  handlers:    
    - name: Install VPN RoutingOnly
      win_shell: >
        Install-RemoteAccess -Legacy -VpnType RoutingOnly
      listen: Configure VPN RoutingOnly
    
    - name: Install VPN VpnS2S
      win_shell: >
        Install-RemoteAccess -Legacy -VpnType VpnS2S
      listen: Configure VPN S2S

    - name: Add new Demand-dial interface 
      win_shell: >
        $mycert = ( Get-ChildItem -Path cert:LocalMachine\My | Where-Object { $_.Subject -match "CN=WIN-SRV-AD.test.dk" } )[0]

        Add-VpnS2SInterface -Name "LIN-RTR.lego.dk" 
        -Destination LIN-RTR.lego.dk
        -Persistent 
        -IPv4Subnet "10.10.0.0/24:10" 
        -Protocol IKEv2 
        -AuthenticationMethod MachineCertificates 
        -Certificate $mycert
      listen: Configure S2S Interface

    - name: Add Certificate to send
      win_shell: >
        $mycert = ( Get-ChildItem -Path cert:LocalMachine\My | Where-Object { $_.Subject -match "CN=WIN-SRV-AD.test.dk" } )[0]

        Set-VpnAuthProtocol  
        -UserAuthProtocolAccepted EAP,MsChapv2,Certificate 
        -TunnelAuthProtocolsAdvertised Certificates 
        -CertificateAdvertised $mycert
      listen: S2S Cert
```

# Inventory

```yaml
all:
  vars:
    s2s:
      debian_subnet: 10.10.0.0/24
      debian_public_ipv4: 20.0.0.2
      windows_subnet: 10.20.0.0/24
      windows_public_ipv4: 20.0.0.3
      tunnel_subnet: 10.255.255.0/24
      winid: '"CN=WIN-SRV-AD.test.dk"'
      ids: 
        - subject: '"CN=WIN-SRV-AD.test.dk"'
          name: '"WIN-SRV-AD.test.dk"'
          filename: "win-s2s.inf"

        - subject: '"CN=LIN-RTR.lego.dk"'
          name: '"LIN-RTR.lego.dk"'
          filename: "lin-s2s.inf"

      files:
        - inf: "lin-s2s.inf"
          req: "lin-s2s.req"
          crt: "lin-s2s.crt"
          pfx: "lin-s2s.pfx"
        
        - inf: "win-s2s.inf"
          req: "win-s2s.req"
          crt: "win-s2s.crt"
          pfx: "win-s2s.pfx"

      fetchfiles:
        - srcname: lin-s2s.pfx
          destname: lin-s2s.pfx
        
        - srcname: CA.crt
          destname: WIN-CA.crt
debian_servers:
  hosts:
    lin_rtr:
      ansible_host: 10.10.0.254

windows_servers:
  hosts:
    WIN-SRV-AD.TEST.DK:
      ansible_host: WIN-SRV-AD.TEST.DK
      ipv4_add: 10.20.0.254
      ansible_connection: winrm
      ansible_winrm_transport: ntlm
      ansible_port: 5985
      ansible_user: Administrator@TEST.DK
      ansible_become_user: TEST\Administrator
      ansible_become_method: runas
      ansible_password: Passw0rd
      ansible_become_password: Passw0rd  

```

# Templates

<kbd>templates/ipsec.conf.s2scert.j2</kbd>

```c
conn rras-cert
    # General settings
    authby=rsasig
    keyexchange=ikev2
    ike=aes256-sha2_256-modp1024
    esp=aes256-sha2_256
    auto=start

    # Debian side settings
    left = {{ s2s.debian_public_ipv4 }}
    leftsubnet = {{ s2s.debian_subnet }}
    leftcert = {{ s2s.cert }}

    # Windows side settings
    right= {{ s2s.windows_public_ipv4 }}
    rightsubnet= {{ s2s.windows_subnet }}
    rightid = {{ s2s.winid }}
    rightsourceip= = {{ s2s.tunnel_subnet }}
```


<kbd>templates/ipsec.secrets.s2scert.j2</kbd>

```c
: RSA {{ s2s.key }}
```