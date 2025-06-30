---
title: ZFS
description: 
published: true
date: 2025-06-30T10:44:05.978Z
tags: linux
editor: markdown
dateCreated: 2025-05-20T12:54:23.739Z
---

# Fundamentals

ZFS is an advanced filesystem with volume management capabilities. The following image describes the basic architecture of ZFS.

![ZFS Components](/zfs_-components.png)

Pools *(zpool)* are composed of VDEVs. VDEVs are striped together to create the pool. Parity/redundancy is handled at the VDEV level.

> There are 'unique' types of redundancy introduced by ZFS, **RAIDZ1** and **RAIDZ2**. These are mostly similar to RAID-5 and RAID-6.
{.is-info}

On top of the pool, you can create 'containers' to hold data. These come in two types:

- **Dataset:** A fully working filesystem where you can put files. A dataset does not have a defined size, it only takes up the space needed.
- **ZVOL:** An empty, block storage volume which can be used just like any disk, e.g. you can create a filesystem on top of it.

# Installing ZFS

Install ZFS by installing the following packages:

```bash
apt install linux-headers-amd64
apt install zfsutils-linux
```

# Configuring ZFS

## Creating a pool

> When working with ZFS, you reference disks by their path/id instead of `/dev/sdaX` because that can change at any restart.
> 
> To see all your drives, run:
> ```bash
> ls -l /dev/disk/by-path/
> ```
> You will see which path points to which drive.
{.is-info}

You can start using ZFS by first creating a pool. In the pool you already define the VDEV(s) it is based on, too. (You can add more VDEVs to the pool later.)

Creating a new pool follows this syntax:

```bash
zpool create <pool_name> <[redundancy] device1 device2 ..> ..
```

For example:

```bash
# Create a pool named 'pool1' from 4 drives with RAIDz1
zpool create pool1 \
  raidz1 \
    pci-0000:03:00.0-scsi-0:0:1:0 \
    pci-0000:03:00.0-scsi-0:0:2:0 \
    pci-0000:03:00.0-scsi-0:0:3:0 \
    pci-0000:03:00.0-scsi-0:0:4:0

# Create a pool named 'pool2' from 4 drives in two VDEVs,
# each mirrored, like in a RAID-10 setup
zpool create pool2 \
  mirror \
    pci-0000:03:00.0-scsi-0:0:1:0 \
    pci-0000:03:00.0-scsi-0:0:2:0 \
  mirror \
    pci-0000:03:00.0-scsi-0:0:3:0 \
    pci-0000:03:00.0-scsi-0:0:4:0
```

> Common redundancy types also include `raidz2`, apart from `raidz1` and `mirror`. You can also omit the redundancy type, thus creating a striped pool. *(Which consists of **x** VDEVs if you use **x** drives.)*
{.is-info}

## Managing datasets

Issue the following commands to create and mount a dataset:

```bash
mkdir /data

# This will create a dataset called 'data' and
# mount it to /data. The mount is automatically
# persisted.
zfs create -o mountpoint=/data pool1/data
```

To destroy a dataset, issue the command:

```bash
# No need to unmount, as the mount is managed by ZFS
zfs destroy pool1/data
```

## Managing volumes

Create a new volume on your pool:

```bash
zfs create -s -V 4GB pool1/volume1
```

This command creates a new volume on `pool1` called `volume1`. The `-s` (sparse) parameter makes it a **thinly provisioned** volume, which means it will only consume the space it needs, instead of taking up 4GB already on creation.

Volumes do not have a filesystem, so they aren't mounted.

```bash
mkfs.ext4 /dev/zvol/pool1/volume1

mkdir /mnt/volume1
mount /dev/zvol/pool1/volume1 /mnt/volume1
```

Because volume mounts aren't managed by ZFS, you will need to umount it yourself if you want to destroy the volume.

```bash
umount /dev/zvol/pool1/volume1
zfs destroy pool1/volume1
```

## Managing snapshots

ZFS comes with the ability to snapshot datasets and volumes. To create a snapshot, use the following command:

```bash
zfs snapshot pool1/data@2025-05-20
```

Rollback to the snapshot:

```bash
zfs rollback pool1/data@2025-05-20
```

Destroy the snapshot:

```bash
zfs snapshot pool1/data@2025-05-20
```

# Checking ZFS

There are many useful commands to check the status of ZFS, some of them are:

```bash
# Shows the summary of your pool configuration
zpool status

# Get the usage of a certain volume/dataset
zfs get all pool1/volume1 | grep used
```

# ZFS encryption

You can create encrypted ZFS pools or datasets.

The easiest way is to use a passphrase for the encryption.

Use the following command to create an encrypted pool:

```bash
zpool create \
  -O encryption=aes-256-gcm \
  -O keyformat=passphrase \
  <pool-name> \
  <vdev> ...
```

> For the encryption type, you can use any of `aes-{128,192,256}-{ccm,gcm}`, or even `on`, which uses the default `aes-128-ccm`.
> 
> The format of **\<vdev>** can be found in the guide above.
{.is-info}

After creating the pool, you can create a dataset on it, and it will be automatically encrypted:

```bash
zfs create <pool-name>/<dataset-name>
```

Check encryption with:

```bash
zfs get encryption <pool-name>/<dataset-name>
```

## Mount after restart

To mount encrypted ZFS pools after a restart, the keys need to be loaded first. In this example, the pool was encrypted, so the keys for that need to be loaded.

```bash
zfs load-key <pool-name>
```

Now you can mount all file systems:

```bash
zfs mount -a
```

### Auto-mount

To automatically mount the share after a restart, write the encryption passhprase into <kbd>/etc/passphrase</kbd>:

```bash
echo '<passphrase>' > /etc/passphrase
chown root:root /etc/passphrase
chmod 600 /etc/passphrase
```

Create <kbd>/etc/rc.local</kbd> and edit it:

```bash
#!/bin/bash
zfs mount -l -a < /etc/passphrase
```

Make the file executable:

```bash
chmod +x /etc/rc.local
```

> This setup is not recommended, but it works if you want to automatically mount ZFS filesystems.
{.is-warning}
