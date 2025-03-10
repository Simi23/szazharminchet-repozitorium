---
title: Apache2
description: Apache2 general configs, mods
published: true
date: 2025-03-10T10:20:34.732Z
tags: linux
editor: markdown
dateCreated: 2025-03-10T10:13:08.895Z
---

# Apache2

## Install

Install apache2 packages

```
apt install apache2
```

## Mods

You can enable mods using the following command.
```
a2enmod modname
```

You can enable mods using the following command.
```
a2dismod modname
```
> You can edit mod configs (if needed) under `/etc/apache2/mods-available`
> 
> Useful mods:
authz_user
ldap
ssl
userdir
> {.is-info}

## Sites
The sites are located under `/etc/apache2/sites-available`. If you add here a .conf file, you can enable it using `a2ensite sitename`. It creates a link into `/etc/apache2/sites-enabled`. You can user `a2dissite sitename` to disable a site.

## Virtual hosting (configs)

> Virtual host for http.
> {.is-info}
```cfg
<VirtualHost *:80>
    ServerName web.company.com

    DocumentRoot /var/www/html
    DirectoryIndex index.html index.php index.html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks # Enable indexing for site
        AllowOverride None
        Require all granted
    </Directory>

    ErrorDocument 404 error.html # Error documents
    Alias /whoami /opt/wwwroot/whoami/main.html # Alias for anything
    
    # Logging
    ErrorLog /var/log/apache2/company/error.log
    CustomLog /var/log/apache2/company/access.log combined
</VirtualHost>
```

> Virtual host for https (you have to enable ssl mod).
> {.is-info}
```cfg
<VirtualHost *:443>
    ServerName web.company.com

		SSLEngine on
    
    SSLCertificateFile /cert/web.crt
    SSLCertificateKeyFile /cert/web.key

    DocumentRoot /var/www/html
    DirectoryIndex index.html index.php index.html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks # Enable indexing for site
        AllowOverride None
        Require all granted
    </Directory>

    ErrorDocument 404 error.html # Error documents
    Alias /whoami /opt/wwwroot/whoami/main.html # Alias for anything
    
    # Logging
    ErrorLog /var/log/apache2/company/error.log
    CustomLog /var/log/apache2/company/access.log combined
</VirtualHost>
```


