---
title: PowerShell NTFS ACLs
description: 
published: true
date: 2025-06-20T08:30:49.570Z
tags: windows, powershell
editor: markdown
dateCreated: 2025-06-20T08:28:25.205Z
---

# Viewing access rules

To get the ACLs of a file/folder, use the following command:

```powershell
$acl = Get-Acl .\location
```

The `$acl.Access` variable will contain all of the access rules related to the object. You can view it by just typing:

```powershell
$acl.Access
```

# Creating new access rules

If you want to add a new access rule, you have to create an object of the **System.Security.AccessControl.FileSystemAccessRule** class.

The constructor of the object takes the following parameters:

- **Identity reference:** Who the ACE is applied to (e.g. `COMPANY\Domain Admins` or `Everyone`)
- **Filesystem rights:** the exact rights the principal has. Most common values:
  - **ReadAndExecute**
  - **Modify**
  - **FullControl**
  - For a **full list**, see the Microsoft [**documentation**](https://learn.microsoft.com/en-us/dotnet/api/system.security.accesscontrol.filesystemrights?view=net-9.0)
- **Inheritance flags:** What kind of child objects can inherit the ACE. Possible values:
  - **None:** No inheritance flags are set
  - **ContainerInherit:** The ACE is inherited by child container objects (folders)
  - **ObjectInherit:** The ACE is inherited by child leaf objects (files)
- **Propagation flags:** How the inheritance is propagated. Possible values:
  - **None:** No inheritance flags are set
  - **NoPropagateInherit:** The ACE is inherited by immediate children only
  - **InheritOnly:** The ACE is propagated only to child objects. (Not applied to the object itself.)
- **Access control type:** Either `Allow` or `Deny`

```powershell
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule( `
  "COMPANY\Domain Admins", `
  "Modify", `
  "ContainerInherit,ObjectInherit", `
  "None", `
  "Allow"
)
```

> You don't need to memorize the object class, just start typing "**FileSystemAccessRule**" and press <kbd>TAB</kbd>.
{.is-info}

After you have created the rule, apply it to the ACL object:

```powershell
$acl.AddAccessRule($rule)
```

When you're done editing the ACL, apply it back to the object:

```powershell
$acl | Set-Acl .\location
```

# Remove access rules

To remove an ACE from an ACL, you need to reference it directly. You can do this by just selecting it by index.

```powershell
$acl = Get-Acl .\location

# List the rules and find which one you want to remove
$acl

$ruleToRemove = $acl.Access[2]

# Double-check if it is the correct one
$ruleToRemove

$acl.RemoveAccessRule($ruleToRemove)

$acl | Set-Acl .\location
```

# Modifying inherited rules

The syntax of `SetAccessRuleProtection(bool, bool)` is the following:

1. **isProtected**
   - `true` to protect the access rules from inheritance (this object will not receive any inherited rules)
   - `false` to allow inheritance
2. **preserveInheritance**
   - `true` to preserve inherited rules as explicit rules
   - `false` to remove inherited access rules
   - This option is ignored if **isProtected** is set to false.

## Convert inherited rules to explicit rules

```powershell
$acl = Get-Acl .\location

$acl.SetAccessRuleProtection($true, $true)

$acl | Set-Acl .\location
```

## Remove all inherited rules

```powershell
$acl = Get-Acl .\location

$acl.SetAccessRuleProtection($true, $false)

$acl | Set-Acl .\location
```
