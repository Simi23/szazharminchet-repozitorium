---
title: One-way file syncing
description: One-way syncing with lsyncd
published: true
date: 2025-06-18T08:06:34.466Z
tags: linux
editor: markdown
dateCreated: 2025-06-18T08:06:34.466Z
---

# Introduction

This guide will demonstrate the configuration of **lsyncd** in order to achieve a **one-way sync** of a folder between **two hosts**.

# Installation

Install lsyncd with the following command:

```bash
apt install lsyncd
```

# Configuration

To configure lsyncd with rsync ssh copy, copy the example file to `/etc`.

```bash
mkdir /etc/lsyncd
cp /usr/share/doc/lsyncd/examples/lrsyncssh.lua /etc/lsyncd/lsyncd.conf.lua
```

Now, edit the variables you need in the file:

```lua
sync { 
  default.rsyncssh,
  source="/path/to/source",
  host="<destination-host>",
  targetdir="/path/to/destination"
}
```

If you need more source/destination pairs, you can copy the sync block.

> Be aware that the source and destination can only be **directories, not individual files.**
{.is-warning}

Because rsync uses ssh to copy files in the background, you need to set up passwordless ssh. In order to do so, enter the following commands on the source computer:

```bash
ssh-keygen
ssh-copy-id <destination-ip>
```

Now, restart and enable lsyncd:

```bash
service lsyncd restart
systemctl enable lsyncd
```