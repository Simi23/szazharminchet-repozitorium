---
title: ES25 - ModB - 1st Solution
description: 
published: true
date: 2025-08-11T08:20:28.090Z
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
- IPv4 settings (`netsh int ipv4 set add Ethernet0 static add mask gateway`, `netsh int ipv4set dns Ethernet0 static dns`)
- IPv6 settings (`netsh int ipv6 set add Ethernet0 add/mask`, `netsh int ipv6 add route ::/0 Ethernet0 gatewayV6`, `netsh int ipv6 set dns Ethernet0 static dnsV6`)
  - Time related settings (`Get-TimeZone`, `Set-TimeZone "Romance Standard Time"`, `Set-Date -Date "08/11/2025 14:35"`)
  
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
  	# OR
  	echo "Passw0rd!" > /ansible/resources/vault_pass
  	echo 'alias ansible-playbook="ansible-playbook --vault-password-file=/ansible/resources/vault_pass"' >> ~/.bashrc	
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
        name: SRV
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
    yGroups: "{{ Users | map(attribute='Department') | unique | list }}"
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
      loop: "{{ yGroups }}"

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
# Directory creator script
function Directory-Create($folder) {
    # Test if $folder path exists
    Write-Host "Test if '$($folder)' folder exists."
    if(!(Test-Item -Path $folder)) {
        New-Item $folder -ItemType Directory | Out-Null
        Write-Host -ForegroundColor Green "'$($folder)' has been created."
    } else {
        Write-Host -ForegroundColor Yellow "'$($folder)' already exists."
    }
}

# Backup-Start string
function Backup-Start($backupString) {
        Write-Host "`r`n================= Backing up $($backupString) =================" -BackgroundColor Black
}

# User backup script
# FirstName, LastName, samAccountName, UserPrincipalName, Email, JobTitle, City, Company, Department, DisplayName, DistinguishedName, HomeDirectory
function User-Backup {
    Backup-Start("Users")

    # Variables
    $csvFile = Join-Path $backupRoot -ChildPath "Users.csv"

    # Logic
    Get-ADUser -Filter * -Properties FirstName, LastName, samAccountName, UserPrincipalName, Email, JobTitle, City, Company, Department, DisplayName, DistinguishedName, HomeDirectory `
    | Select-Object FirstName, LastName, samAccountName, UserPrincipalName, Email, JobTitle, City, Company, Department, DisplayName, DistinguishedName, HomeDirectory `
    | Export-Csv -Path $csvFile -NoTypeInformation -Encoding UTF8

    Write-Host "User Backup DONE!" -ForegroundColor Green
}

# GPO backup script
function GPO-Backup {
    Backup-Start("GPOs")

    # Variables
    $gpoPath = Join-Path $backupRoot -ChildPath "GPO"
    $gpoDeletePath = Join-Path $gpoPath -ChildPath "*"

    # Logic
    Directory-Create($gpoPath)
    Remove-Item $gpoDeletePath -Recurse -Force | Out-Null

    Backup-GPO -All -Path $gpoPath | Out-Null
    
    Write-Host "GPO Backup DONE!" -ForegroundColor Green
}

# IIS webroot backup script
function IIS-Backup {
    Backup-Start("IIS Webroots")

    # Variables
    $iisPath = Join-Path $backupRoot -ChildPath "Web"
    $iisServers = @("SRV2.skillsnet.dk", "DC.skillsnet.dk")
    $password = ConvertTo-SecureString "Passw0rd!" -AsPlainText -Force
    $credentials = New-Object pscredential("SKILLSNET\Administrator", $password)
    
    # Logic
    foreach ($iisServer in $iisServers) {
        Write-Host "Connecting to host $($iisServer)"
        $session = New-PSSession -ComputerName $iisServer -Credential $credentials
        $sites = Invoke-Command -ComputerName $iisServer -Credential $credentials -ScriptBlock { "Get-WebSite" }

        foreach ($site in $sites) {
            $siteName = $site.Name
            $sitePath = $site.PhysicalPath.Replace("%SystemDrive%", "C:")
            $sitePath = Join-Path $sitePath -ChildPath "*"
            $localPath = Join-Path $iisPath -ChildPath $siteName

            Write-Host "Backing up '$($siteName)' webroot."

            Copy-Item -Path $sitePath -Destination $localPath -FromSession $session -Recurse -Force | Out-Null

            Write-Host "DONE!" -ForegroundColor Green
        }
    }

    Write-Host "IIS WebRoot Backup DONE!" -ForegroundColor Green
}


# Variables
$backupRoot = "C:\Backups"

# Running functions
try {
    Directory-Create($backupRoot)
    User-Backup
    GPO-Backup
    IIS-Backup
    $subject = "Backup success"
    $body = "Backup successfully ran on $($env:COMPUTERNAME).skillsnet.dk at $(Get-Date)." 
    Send-MailMessage -SmtpServer "mail.nordicbackup.net" -To "support@nordicbackup.net" -From "backup@skillsnet.dk" -Body $body -Subject $subject
} catch {
    Write-Host "ERROR: $($_.error.message)" -ForegroundColor Red
    $subject = "Backup failure"
    $body = "Backup failure on $($env:COMPUTERNAME).skillsnet.dk at $(Get-Date).`r`nERROR: $($_.error.message)" 
    Send-MailMessage -SmtpServer "mail.nordicbackup.net" -To "support@nordicbackup.net" -From "backup@skillsnet.dk" -Body $body -Subject $subject
}
  ```
</details>

[//]: <> (Bitlocker)
<details>
<summary>Bitlocker</summary>
 
- `Install-WindowsFeature Bitlocker -IncludeManagementTools`
- `Restart-Computer`
- `Enable-BitLocker -TpmProtection "C:\`
- `$password = ConvertTo-SecureString "Passw0rd!" -AsPlainText -Force`
- `Enable-BitLocker -PasswordProtection "D:\" -Passw0rd $password`
- `Enable-BitLockerAutoUnlock "D:\"
> Bitlocker TPM encryption doesn't work in anything else than system drive, if there are snapshots on a VM or it has ThinProvision disk, in this build of the Windows 2022 you can not use BitLocker encryption!
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
> `Enable-NetFirewallRule -DisplayGroup "Remote File Server Resource Manager Management` on CORE server.
{.is-info}

</details>


[//]: <> (DNS)
<details>
<summary>DNS</summary>

  > **+ CNAME Records to add**
  > <span>DC.skillsnet.</span>dk: **sso**, **ocsp**
  > <span>SRV2.skillsnet.</span>dk: **app**, **cacerts**, **crl**, **intra**, **www**
  > <span>DEV-SRV.skillsdev.</span>dk: **www**
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
>   - If it still isn't working restart both service and don't restart the computer!
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
  > When creating a new share on SRV1 & SRV2 follow these scheme.
  > {.is-info}
  `New-SmbShare D:\Users -Name 'Users' -EncrypData $true -FullAccess 'Domain Users' -ReadAccess 'Everyone'` 
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
> -  Add `<ConfigurationMode> Custom </ConfigurationMode>` to the first line
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
> - `Enable-NetFirwallRule -DisplayGroup 'Remote Event Log Management'`
>
> _
{.is-info}

> DON'T MESS UP THE GPOS, BECAUSE CACHE REAMINS, AND YOU CAN JUST IMAGINE ABOUT THE 100%!
{.is-danger}

</details>




