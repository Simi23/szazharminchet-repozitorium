---
title: Quota
description: Quota
published: true
date: 2025-06-03T12:45:14.983Z
tags: 
editor: markdown
dateCreated: 2025-06-03T12:45:14.983Z
---

# Quota

Install the following package `apt install quota.

> Edit /etc/fstab and write this to the 'option' part of the specific filesystem:
{.is-info}


```bash
usrquota,grpquota
```

> Remount the filesystem:
{.is-info}

```bash
mount -o remount <mount point>
```

> Create the necessary user and group quota files:
{.is-info}

```bash
quotacheck -cugm <mount point>
```

> Enable the quota:
{.is-info}

```bash
quotaon <mount point>
```

> Edit users' or groups' quota (values in KB):
{.is-info}

```bash
edquota -u <username>
edquota -g <groupname>
```
Or:
```bash
setquota -u <username> <block-soft> <block-hard> <inode-soft> <inode-hard> <mount point>
setquota -g <groupname> <block-soft> <block-hard> <inode-soft> <inode-hard> <mount point>
```
# Optional
> Edit the grace time:
{.is-info}
```bash
edquota -t
```
> Copy quota of an user to other:
{.is-info}
```bash
edquota -p <from user> <to user> <to user2>
```
> Check quota usage of an user:
{.is-info}
```bash
quota -u <username>
```
> Check all quotas on a filesystem:
{.is-info}
```bash
repquota <mount point>
```
> Disable quota on a filesystem:
{.is-info}
```bash
quotaoff <mount point>
```
> Sending a mail in case of exceeding the soft limit, or expiry of grace time:
{.is-info}
Its parameters are in /etc/warnquota.conf
```bash
warnquota
```




