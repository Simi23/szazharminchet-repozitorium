---
title: Icinga2 install
description: 
published: true
date: 2025-06-18T07:50:23.620Z
tags: linux
editor: markdown
dateCreated: 2025-05-29T09:44:50.187Z
---

# Icinga2 setup

## Installing and setting up from CLI
> Install the following packages:
{.is-info}

```
apt install apache2 libapache2-mod-php icinga2 icingadb icingadb-web redis-server monitoring-plugins mariadb-server icingaweb2
```

> Enable following services
{.is-info}
```
systemctl enable icingadb redis-server
```

> Run the api setup
{.is-info}

```
icinga2 api setup
icinga2 feature enable icingadb
```

> Enter mySQL cli, and create database, and following settings:
{.is-info}

```
mysql -u root -p
```

```sql
CREATE DATABASE icingadb;
GRANT ALL ON icingadb.* TO 'icingadb'@'localhost' IDENTIFIED BY 'Passw0rd';
FLUSH PRIVILEGES;
quit;
```

> Add the schema into the database
{.is-info}

```
mysql -u root -p icingadb </usr/share/icingadb/schema/mysql/schema.sql
```

> Edit icingadb settings `/etc/icingadb/config.yml`. Change the password what you have given as mysql user password, and edit redis port to 6379. Restart the service.
{.is-info}

```
systemctl restart icingadb
```

> Complete some actions so you can finish your setup with icingaweb
{.is-info}

```
chmod -R 775 /var/lib/icingaweb2
chmod -R 775 /etc/icingaweb2
chown -R root:icingaweb2 /var/lib/icingaweb2
chown -R root:icingaweb2 /etc/icingaweb2
usermod -aG icingaweb2 www-data
```

> For the web setup you will need these credentials, so make some notes about these
{.is-info}

```
cat /etc/icinga2/conf.d/api-users.conf
icingacli setup token create
```

## Installing and setting up from WEB

> Open a web browser and follow these steps:
{.is-info}

` - access the web gui under http://ip/icingaweb2/setup`
` - when it asks for a token then paste it in from the prompt you got from the icingacli command`
 - check in just icingadb
 - when it asks for any database you have to use icingadb as name your password as password and 3306 as port, click check before continue
 - when it asks for user and password use the credentials from /etc/icinga2/conf.d/api-users.conf
 - as primary redis host set localhost and port 6379 if you did not change it
 - create an admin user to access the web gui later, and send emails to them

> You're now all done, you can configure Icinga2 monitoring from the configuration files.
{.is-info}
