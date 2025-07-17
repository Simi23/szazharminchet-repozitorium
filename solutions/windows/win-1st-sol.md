---
title: ES25 - ModB - 1st Solution
description: 
published: true
date: 2025-07-17T09:07:00.650Z
tags: windows, es25-windows, es25
editor: markdown
dateCreated: 2025-06-26T09:03:28.237Z
---

# ES25 - ModB - 1st Solution
![modb-tasks.jpg](/solutions/assets/modb-tasks.jpg)

[//]: <> (General)
<details>
<summary>General</summary>

- Hostname (`Rename-Computer -Name HOSTNAME`)
- IPv4 settings (`netsh int ipv4 set add Ethernet0 static add mask gateway`)
- IPv6 settings (`netsh int ipv6 set add Ethernet0 add/mask`)
  
</details>

[//]: <> (AD CS)
<details>
<summary>AD CS</summary>
  
</details>

[//]: <> (AD DS)
<details>
<summary>AD DS</summary>
  
  **DC settings**
  - `Install-WindowsFeature -Name Ad-Domain-Services, DNS -IncludeManagementTools`
  - `$password = ConvertTo-SecureString -AsPlainText -Force "Passw0rd!"`
  - `Install-ADDSForest -DomainName skillsnet.dk -SafeModePassword $password`
  
  **RODC settings**
  - DNS settings
  - `Add-Computer -DomainName skillsnet.dk`
  - `Restart-Computer`
  - `$password = ConvertTo-SecureString -AsPlainText -Force "Passw0rd!"`
  - `Install-WindowsFeature -Name Ad-Domain-Services, DNS -IncludeManagementTools`
  - `Install-ADDSDomainController -DomainName skillsnet.dk -SiteName Default-First-Site -SafeModePassword $password`
  
  **CLIENT settings**
  - DNS settings
  - `Add-Computer -DomainName skillsnet.dk`
  - `Restart-Computer`
  
</details>

[//]: <> (AD FS)
<details>
<summary>AD FS</summary>
  
</details>

[//]: <> (AD Sites)
<details>
<summary>AD Sites</summary>

>   DO IT LAST AND DON'T FORGET IT
{.is-warning}
</details>


[//]: <> (Ansible)
<details>
<summary>Ansible</summary>
  
  Create ansible vault
  
  ```bash
  	echo "export EDITOR=nano" >> ~/.bashrc	
  	echo 'alias ansible-playbook="ansible-playbook --ask-vault-password"' >> ~/.bashrc	
    source ~/.bashrc
  	ansible-vault create /ansible/resources/vault.yml
  ```
  
  <kbd>1-hostname.yaml</kbd>
  
  ```yaml
  ---
- name: Hostname
  hosts: all
  gather_facts: false
  tasks:
    # | Change hostname | 
    - name: Change hostname
      ansible.windows.win_hostname:
        name: "{{ hostname }}"
      register: reg
      notify: Reboot

  handlers:
    # | Reboot |
    - name: Reboot
      ansible.windows.win_reboot:
      when: reg.reboot_required
  ```
  
  
  <kbd>2-adds.yaml</kbd>
  
  ```yaml
  ---
- name: ADDS
  hosts: all
  gather_facts: false
  vars_files:
    - resources/vault.yml
  tasks:
    # | Install ADDS | 
    - name: Install ADDS
      ansible.windows.win_feature:
        name: 
          - Ad-Domain-Services
          - DNS
        state: present
      
    # | Deploy ADDS |
    - name: Deploy ADDS
      microsoft.ad.domain:
        dns_domain_name: skillsdev.dk
        safe_mode_password: "{{ secret_password }}"
        reboot: true
  ```
  
  <kbd>3-users.yaml</kbd>
  
  ```yaml
  ---
- name: OU and User creation
  hosts: all
  gather_facts: false
  become: true
  vars:
    OUs: "{{ lookup('file', 'resources/OU.json') | from_json }}"
    Users: "{{ lookup('file', 'resources/ES2025_TP39_ModuleB_Users_Skillsdev.json') | from_json }}"
    Groups: "{{ Users | map(attribute='Department') | unique | list }}"
  vars_files:
    - resources/vault.yml
  tasks:
    # | Create OU structure |
    - name: Create OU structure
      microsoft.ad.ou:
        name: "{{ item.OU }}"
        path: "{{ item.Path }}DC=skillsdev,DC=dk"
        description: "{{ item.Description }}"
        state: present
      loop: "{{ OUs }}"
      loop_control:
        label: "{{ item.OU }}"

    # | Create groups |
    - name: Create groups
      microsoft.ad.group:
        name: "{{ item }}"
        scope: global
        path: "OU=Groups,OU=Skills,DC=skillsdev,DC=dk"
        state: present
      loop: "{{ Groups }}"

    # | Create users |
    # "FirstName": "Jill",
    # "LastName": "Santiago",
    # "Email": "jill.santiago@skillsdev.dk",
    # "JobTitle": "Insurance account manager",
    # "City": "Catherineton",
    # "Company": "Skillsdev",
    # "Department": "Tech"
    - name: Create users
      microsoft.ad.user:
        name: "{{ item.FirstName }} {{ item.LastName }}"
        firstname: "{{ item.FirstName }}"
        surname: "{{ item.LastName }}"
        email: "{{ item.Email }}"
        city: "{{ item.City }}"
        company: "{{ item.Company }}"
        password: "{{ secret_password }}"
        sam_account_name: "{{ item.FirstName }}.{{ item.LastName }}"
        upn: "{{ item.FirstName }}.{{ item.LastName }}@skillsdev.dk"
        path: "OU={{ item.Department }},OU=Users,OU=Skills,DC=skillsdev,DC=dk"
        update_password: on_create
        groups:
          set:
            - "{{ item.Department }}"
            - "Domain Users"
        attributes:
          set:
            Title: "{{ item.JobTitle }}"
            Department: "{{ item.Department }}"
        state: present
      loop: "{{ Users }}"
      loop_control:
        label: "{{ item.FirstName }}.{{ item.LastName }}"
  	```
  
  <kbd>4-web.yaml</kbd>
  
  ```yaml
---
- name: IIS Configruation
  hosts: all
  gather_facts: false
  become: true
  tasks:
    # | Install IIS |
    - name: Install IIS
      ansible.windows.win_feature:
        name: Web-Server

    # | Copy IIS Website |
    - name: Copy IIS Website
      ansible.windows.win_copy:
        dest: C:\inetpub\wwwroot\iisstart.htm
        content: "<h1>Skills Development</h1>"
        
    # | Create DNS record for webserver |
    - name: Create DNS record for webserver
      community.windows.win_dns_record:
        name: "www"
        type: "CNAME"
        value: "DEV-SRV.skillsdev.dk"
        zone: "skillsdev.dk"
        
  ```
  
  
  <kbd>5-shares.yaml</kbd>
  
  ```yaml
---
- name: Create CIFS Shares
  hosts: all
  gather_facts: false
  become: true
  vars_files:
    - resources/ES2025_TP39_ModuleB_Shares.yaml
  tasks:
    # | Create directories |
    - name: Create directories
      ansible.windows.win_file:
        path: "{{ item.path }}"
        state: directory
      loop: "{{ shares }}"
      loop_control:
        label: "{{ item.name }}"

    # | Create CIFS Shares |
    - name: Create CIFS Shares
      ansible.windows.win_share:
        name: "{{ item.name }}"
        path: "{{ item.path }}"
        description: "{{ item.description }}"
        full_access: "{{ item.full_access }}"
        read_access: "{{ item.read_access }}"
        state: present
      loop: "{{ shares }}"
      loop_control:
        label: "{{ item.name }}"
  ```
  
  
> CREATE AND USE THE JSON
{.is-warning}
</details>

[//]: <> (Backup)
<details>
<summary>Backup</summary>
  
> USE COMMENTS AND ADD COMMENTS TO YOUR OUTPUT TOO
{.is-warning}
  ```powershell
# Variables
# Emails settings
$smtpServer = "mail.nordicbackup.net"
$from = "backup@skillsnet.dk"
$to = "support@nordicbackup.net"
$subject = ""
$body = ""
$success = $true

# Backup path
$BackupRoot = "C:\Backups"
$usersCSV = "$backupRoot\Users.csv"
$gpoBackup = "$backupRoot\GPOs"
$webBackup = "$backupRoot\Web"

try {
    # Create backup folders
    Write-Output "Create backup folders"
    New-Item -Path $BackupRoot -ItemType Directory -Force | Out-Null
    New-Item -Path $gpoBackup -ItemType Directory -Force | Out-Null
    New-Item -Path $webBackup -ItemType Directory -Force | Out-Null

    # Export users to CSV
    # FirstName,LastName,samAccountName,UserPrincipalName,Email,JobTitle,City,Company,Department
    Write-Output "Export users to CSV"
    Get-ADUser -Filter * -Properties GivenName, Surname, SamAccountName, DistinguishedName, UserPrincipalName, EmailAddress, Title, City, Company, Department `
        | Select-Object GivenName, Surname, SamAccountName, DistinguishedName, UserPrincipalName, EmailAddress, Title, City, Company, Department `
        | Export-Csv -Path $usersCSV -NoTypeInformation -Encoding UTF8


    # Export Group Policy Objects
    Write-Output "Export Group Policy Objects"
    $allGPO = Get-GPO -All
    foreach ($gpo in $allGPO) {
        $gpoName = $gpo.DisplayName
        $gpoPath = Join-Path $gpoBackup $gpoName
        New-Item -Path $gpoPath -ItemType Directory -Force | Out-Null
        Backup-GPO -Name $gpoName -Path $gpoPath -ErrorAction SilentlyContinue
    }

    # Backup IIS web root folders
    Write-Output "Backup IIS web root folders"
    Import-Module WebAdministration

    # Get all IIS sites
    $sites = Get-Website

    foreach ($site in $sites) {
        $siteName = $site.Name
    
        $sourcePath = $site.PhysicalPath
        $sourcePath = $sourcePath.Replace('%SystemDrive%', 'C:')
        $destinationPath = "C:\Backups\Web\$siteName"

        Write-Host "Backing up site '$siteName' from $sourcePath to $destinationPath"

        # Create destination folder
        New-Item -ItemType Directory -Path $destinationPath -Force | Out-Null

        # Copy the site files
        Copy-Item -Path $sourcePath\ -Destination $destinationPath -Recurse -Force -ErrorAction Stop
    }

    
    # Copy items to iSCSI disk
    Copy-Item -Path $BackupRoot -Destination "b:\" -Recurse -Force
    
    # Send success email
    Write-Output "Sending success email"
    $subject = "Backup Success on $env:COMPUTERNAME"
    $body = "The backup completed successfully on $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')."
    Send-MailMessage -From $from -To $to -Subject $subject -Body $body -SmtpServer $smtpServer
} catch {
    # Sending failure email
    Write-Output "Sending failure email"
    $succes = $false
    $subject = "Backup FAILED on $env:COMPUTERNAME"
    $body = "Backup failed on $(Get-Date). Error: $($_.Exception.Message)"
    Send-MailMessage -From $from -To $to -Subject $subject -Body $body -SmtpServer $smtpServer
    Write-Output "Backup failed on $(Get-Date). Error: $($_.Exception.Message)"
}
  ```
</details>

[//]: <> (Bitlocker)
<details>
<summary>Bitlocker</summary>
 
- `Install-WindowsFeature Bitlocker -IncludeManagementTools`
- `Enable-Bitlocker -TpmProtection "D:\"`
> Bitlocker TPM encryption doesn't work in anything else than system drive
{.is-danger}

</details>

[//]: <> (DFS + FSRM)
<details>
<summary>DFS + FSRM</summary>
  
  - `Install-WindowsFeature FS-Resource-Manager, FS-DFS-Namespace, FS-DFS-Replication -IncludeManagementTools`
  - `Enable-NetFirewallRule -DisplayGroup "Remote File Server Resource Manager Management"`

> **DFS**
> Create the NAMESPACE and it will configure the Replication for you
{.is-info}

  
> **FSRM**
> Do it from Management console, it will be faster.
{.is-info}

</details>


[//]: <> (DNS)
<details>
<summary>DNS</summary>

  > **+ CNAME Records to add**
  > <span>DC.skillsnet.</span>dk: **sso**, **ocsp**
  > <span>SRV2.skillsnet.</span>dk: **app**, **cacerts**, **crl**, **intra**, **www**
  {.is-info}

</details>

[//]: <> (IIS)
<details>
<summary>IIS</summary>
  
</details>

[//]: <> (iSCSI)
<details>
<summary>iSCSI</summary>
  
>   **Target**
>   - Add from server manager and get done everyting with the server manager
>   - After done with settings Restart **WinTarget** and set it's *startup type* to *automatic*
>   - Start **MSiSCSI** and set it's *startup type* to *automatic* 
{.is-info}

  
>   **Initiator**
>   - Start **MSiSCSI** and set it's *startup type* to *automatic*
>   - Connect from iSCSI Initiatior management console (from tools)
{.is-info}

</details>


[//]: <> (Firewall)
<details>
<summary>Firewall</summary>
  
</details>

[//]: <> (GPO)
<details>
<summary>GPO</summary>

  > DO THE PASSWORD POLICICES
{.is-warning}

</details>

[//]: <> (HA DHCP)
<details>
<summary>HA DHCP</summary>
  
</details>

[//]: <> (PS)
<details>
<summary>PS</summary>
  
> CREATE THE JSON FOR OU STRUCTURE
{.is-warning}
  ```powershell
  $json_path = "C:\Resources\OUs.json"
$json = Get-Content -Raw $json_path | ConvertFrom-Json 

Write-Host "============= Creating OUs ============="  -BackgroundColor Black -ForeGroundColor White
foreach ($ou in $json) {
    $newPath = "OU=$($ou.Name),$($ou.Path)DC=skillsnet,DC=dk"
   
    if (Get-ADOrganizationalUnit -Filter { distinguishedName -eq $newPath }) {
        Write-Host "$($ou.Name) OU already exists!" -ForeGroundColor Green -BackgroundColor Black
    } else {
        New-ADOrganizationalUnit -Name $ou.Name -Path "$($ou.Path)DC=skillsnet,DC=dk" -Description $ou.Description -ProtectedFromAccidentalDeletion $false -ErrorAction SilentlyContinue | Out-Null
        Write-Host "$($ou.Name) OU has been created successfully!" -ForeGroundColor Green -BackgroundColor Black
    }
}

$csv_path = "C:\Resources\ES2025_TP39_ModuleB_Users_Skillsnet.csv"
$csv = Import-Csv $csv_path
$password = ConvertTo-SecureString -AsPlainText -Force "Passw0rd!Passw0rd!!!!"
$i = 1
$groups = $csv | Select-Object -ExpandProperty Department | Sort-Object -Unique

Write-Host "`r`n`r`n============= Creating Groups ============="  -BackgroundColor Black -ForeGroundColor White

foreach ( $group in $groups ) {
	$exGroup = Get-ADGroup -Filter { Name -eq $group } -SearchBase "OU=Groups,OU=Skills,DC=skillsnet,dc=dk" -ErrorAction SilentlyContinue

    if (!$exGroup) {
        New-ADGroup -Name $group -Path "OU=Groups,OU=Skills,DC=skillsnet,dc=dk" -GroupScope Global
	    Write-Host "$group group has been created successfully!" -ForeGroundColor Green -BackgroundColor Black
    } else {
        Write-Host "$group group already exists!" -ForeGroundColor Green -BackgroundColor Black
    }
}


Write-Host "`r`n`r`n============= Creating Users ============="	 -BackgroundColor Black -ForeGroundColor White
# FirstName,LastName,samAccountName,UserPrincipalName,Email,JobTitle,City,Company,Department
# Kell siminek Display-name (funame)
foreach ($user in $csv) {
    $finame = $user.FirstName
    $laname = $user.LastName
    $funame = $user.FirstName + " " + $user.LastName
    $sam = $user.Firstname + "." + $user.LastName
    $upname = $user.UserPrincipalName
    $mail = $user.Email
    $title = $user.JobTitle
    $city = $user.City
    $company = $user.Company
    $group = $user.Department

    $exUser = Get-ADUser -Filter { SamAccountName -eq $sam } -ErrorAction SilentlyContinue

    if ($exUser) {
        Write-Host "$i. The $sam user exists." -ForeGroundColor Green -BackgroundColor Black
    } else {
        New-ADUser -Path "OU=$group,OU=Users,OU=Skills,DC=skillsnet,DC=dk" `
            -Name $funame `
            -Enabled $true `
            -AccountPassword $password `
            -GivenName $laname `
            -SurName $laname `
            -DisplayName $funame `
            -UserPrincipalName $upname `
            -SamAccountName $sam `
            -EmailAddress $mail `
            -Title $title `
            -City $city `
            -Company $company `
            -Department $group
            
    
        Add-ADGroupMember -Identity $group -Members $sam
        Write-Host "$i. User $sam has been created and added to $group group!" -ForeGroundColor Green -BackgroundColor Black
    }

    $i++
}
Write-Host "============= Users and groups have been created! =============" -BackgroundColor Black -ForeGroundColor White
  ```
</details>

[//]: <> (S2S - PSK)
<details>
<summary>S2S (PSK)</summary>
  
</details>

[//]: <> (S2S - Cert)
<details>
<summary>S2S (Cert)</summary>
  
</details>

[//]: <> (SMB)
<details>
<summary>SMB</summary>
  `Set-SmbServerConfiguration -EncryptData $true -RejectUnencryptedAccess $true`
</details>

[//]: <> (WAP)
<details>
<summary>WAP</summary>
  
</details>

[//]: <> (WEF)
<details>
<summary>WEF</summary>

  > **GPO**
  > - Computer > Policies > Windows > Security > Restrict Groups > Event Log Readers==> NETWORK SERVICE
  > 
  > - Computer > Policies > Windows > Security > System Services > WinRM (AutoStart)
  > 
  > - Computer > Policies > ADMX > Windows Components > Event Forwarding > Subscription Manager (Server=https://SRV2.skillsnet.dk:5986/wsman/SubscriptionManager/WEC,Refresh=60)
  > 
  > - Computer > Policies > ADMX > Windows Components > Event Log Service > Security > Configure Log Access (`O:BAG:SYD:(A;;0xf0005;;;SY)(A;;0x5;;;BA)(A;;0x1;;;S-1-5-20)(A;;0x1;;;S-1-5-32-573)`)
 >
 > _
{.is-info}

> **SUBSCRIPTION**
> - Start an **Event Viewer**, create a new Subscription
> - `wecutil gs "Subscription Name" /f:xml
> - **Copy** the output, **transfer** it to the CORE computer
> - Disable **wecsvc**!
>
> _
{.is-info}

  
> **SRV2**
> - `gpupdate /force` (Get the computer auto-enrollment Certificate)
> - `winrm qc -transport:https`
> - `wecutil qc`
> - `wecutil cs ./log.xml` (the file you transferred)
> - `Start-Service wecsvc`
> - `Set-Service wecsvc -StartupType Automatic`
> - `Enable-NetFirwallRule -DisplayGroup Remote Event Log Management`
>
> _
{.is-info}

> DON'T MESS UP THE GPOS, BECAUSE CACHE REAMINS, AND YOU CAN JUST IMAGINE ABOUT THE 100%!
{.is-danger}

</details>




