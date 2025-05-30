---
title: Zabbix Setup
description: 
published: true
date: 2025-05-30T13:36:04.362Z
tags: linux
editor: markdown
dateCreated: 2025-05-28T18:35:50.133Z
---

# Zabbix Setup Guide

This guide will demonstrate the installation of **Zabbix 6.0** from the Debian repo.

## Package installation

To start, **install** all required packages:

```bash
apt install \
  apache2 libapache2-mod-php php-mysql \
  zabbix-server-mysql zabbix-frontend-php zabbix-agent zabbix-web-service \
  snmp
```

## Database setup

Installing the package `zabbix-server-mysql` will install a **MariaDB** instance. This will need to be set up in order to be used by Zabbix. This includes importing the schema and initial data.

> There is a file `/usr/share/doc/zabbix-server-mysql/README.Debian` get rid off the unneccessary lines, add a shebang to the start of the file and give it execute permission, after that you can run it as a script.
{.is-info}

```
nano /usr/share/doc/zabbix-server-mysql/README.Debian
chmod +x /usr/share/doc/zabbix-server-mysql/README.Debian
/usr/share/doc/zabbix-server-mysql/README.Debian
```

### OR

**Enter** the MariaDB server.

```bash
mysql -u root -p
```

**Create** a new database and a user. You will also need to set a global to make sure the imports work correctly.

```sql
CREATE DATABASE zabbix CHARACTER SET utf8 COLLATE utf8_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'Passw0rd';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;

SET GLOBAL log_bin_trust_function_creators = 1;
```

> The **last command** is crucial, **do not omit** it by any means, otherwise, importing the initial data will **fail!**
{.is-danger}

Now, **exit MariaDB** and import the SQL files.

```bash
cd /usr/share/zabbix/zabbix-server-mysql
zcat schema.sql.gz | mysql --default-character-set=utf8mb4 -u zabbix -p zabbix
zcat images.sql.gz | mysql --default-character-set=utf8mb4 -u zabbix -p zabbix
zcat data.sql.gz | mysql --default-character-set=utf8mb4 -u zabbix -p zabbix
```

Now you can turn off that global variable.

```bash
mysql -u root -p
```

```sql
SET GLOBAL log_bin_trust_function_creators = 0;
```

## Zabbix configuration

Next, **edit** the file <kbd>/etc/zabbix/zabbix_server.conf</kbd>, find the following lines and change the values:

```ini
DBName=zabbix
DBUser=zabbix
DBPassword=Passw0rd
AllowUnsupportedDBVersions=1
```

**Restart** Zabbix.

```bash
systemctl restart zabbix-server
```

**Enable** the Apache2 config for Zabbix.

```bash
a2enconf zabbix-frontend-php.conf
```

This will provide you access to your Zabbix instance at `http://<ip>/zabbix`. If you don't want to navigate to this subdirectory, you can append the default vHost config with the following line to redirect requests from `/` to `/zabbix`:

```ini
RedirectMatch ^/$ /zabbix/
```

You will need to edit some PHP variables, too. **Edit** the file <kbd>/etc/php/8.2/apache2/php.ini</kbd>.

```ini
post_max_size 16M
max_execution_time 300
max_input_time 300
```

> If you forget these, the Zabbix installer will tell you.
{.is-info}

Now, **restart** Apache2.

```bash
systemctl restart apache2
```

## Running the wizard

To finish installing your Zabbix instance, you will need to **complete the wizard**.

In your browser, **navigate** to `http://<zabbix-instance-ip>/zabbix`.

When configuring the database, **select MySQL** and fill out all details according to your configuration.

> When complete, the wizard will generate a configuration file and it will try to replace the original file, but **this might fail**. In this case, **download** the new config and **replace the file** according to the instructions provided by the wizard.
{.is-warning}

After this, **restart** Zabbix.

```bash
systemctl restart zabbix-server
```

## Accessing your instance

After the installation wizard is complete, you can access Zabbix with the following **default credentials**:

**Username:** `Admin`
**Password:** `zabbix`
