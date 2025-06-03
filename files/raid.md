---
title: RAID
description: Software RAID
published: true
date: 2025-06-03T12:26:35.347Z
tags: linux
editor: markdown
dateCreated: 2025-06-03T12:24:19.740Z
---

# RAID

Install the following package `apt install mdadm`.

> Create a new raid block:
{.is-info}


```bash
mdam --create /dev/md0 --level=x --raid-devices=2 /dev/sda /dev/sdb
mdam --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u
```

> Create the files system and create a directory for the raid block:
{.is-info}

```bash
mkfs.ext4 /dev/md0
mkdir -p /mnt/md0
```

> Edit `/etc/fstab`  to mount the raid block automatically. You can get the uuid with (and add to the eof) `lsblk -i name,uuid | grep /dev/md0 >> /etc/fstab`
{.is-info}

```bash
UUID="(Piped part)" /mnt/md0 auto defaults,nofail,discard 0 0
```

> Try to mount everything from `/etc/fstab`, to see the raid block can be mounted and working.
{.is-info}

```bash
mount -a
systemctl daemon-reload
```