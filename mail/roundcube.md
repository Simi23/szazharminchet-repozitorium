---
title: Roundcube
description: Webmail setup with Roundcube
published: true
date: 2025-02-13T07:56:36.518Z
tags: linux
editor: markdown
dateCreated: 2025-02-12T18:59:21.943Z
---

# Roundcube

You have to install `php`, `apache2/nginx`, `mariadb`, and `roundcube` packages using apt.

Install packages for `php`:

```
apt install php8.2 php8.2-xml php8.2-intl php8.2-curl php8.2-gd php8.2-imagick php8.2-zip php8.2-imap php8.2-smtp php8.2-mysql php8.2-mbstring
```

## Webserver setup

### Nginx

Install packages for `nginx`


Edit `/etc/nginx/sites-available/roundcube` file.

```
server {  
    listen 80;  
    
    server_name roundcube.company.com;  
    root /var/lib/roundcube;  
    index index.php index.html;  
    
    location / {  
        try_files $uri $uri/ /index.php;  
    }  
    
    location ~ \.php$ {  
        include snippets/fastcgi-php.conf;  
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;  
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;  
        include fastcgi_params;  
    }  
    
    location ~ /\.ht {  
        deny all;  
    }  
}  
```

Enable the site and restart the daemon.

```
ln -s /etc/nginx/sites-available/roundcube /etc/nginx/sites-enabled/  
systemctl restart nginx  
```

### Apache2


Install packages for `apache2`:

```
apt install apache2 apache2-utils libapache2-mod-php libapache2-mod-php8.2
```

You have to configure a virtual host for your roundcube server. The files for the website are stored in `/usr/lib/roundcube` folder.


Edit `/etc/apache2/sites-available/roundcube.conf` file:
```
<VirtualHost *:80>  
    ServerName roundcube.company.com 
    DocumentRoot /var/lib/roundcube  
    
    <Directory /var/lib/roundcube>  
        Options FollowSymLinks  
        AllowOverride All  
        Require all granted  
    </Directory>  
    
    ErrorLog ${APACHE_LOG_DIR}/roundcube_error.log  
    CustomLog ${APACHE_LOG_DIR}/roundcube_access.log combined  
</VirtualHost>
```

Enable the site, diasble the default site and restart the daemon.

```
a2ensite roundcube.conf
a2dissite 000-default.conf

systemctl restart apache2
```


## Mariadb

Packages for `mariadb`:

```
apt install mariadb-server mariadb-client
```

Secure the mariadb installation.

```
mysql_secure_installation  
```

Create database for roundcube.

> If you install using apt you don't have to do this step. At install of roundcube you have to set a password for the roundcube sql user, and it will set everything for you.
{.is-info}

```
mysql -u root -p  
```

```SQL
CREATE DATABASE roundcubemail;  
CREATE USER 'roundcubeuser'@'localhost' IDENTIFIED BY 'CHANGEME';  
GRANT ALL PRIVILEGES ON roundcubemail.* TO 'roundcubeuser'@'localhost';  
FLUSH PRIVILEGES;  
EXIT;  
```

## Final roundcube config

Packages for `roundcube`:

```
apt install roundcube
```

Edit the `/etc/roundcube/config.inc.php` file. *(The config lines are in the file yet. This is the minimum config, if you need further check `/etc/roundcube/defaults.inc.php` file.)*

```php
<?php
$config = [];

include("/etc/roundcube/debian-db-roundcube.php");

// Imap server
$config['imap_host'] = 'tls://imap.company.com:143';

$config['smtp_host'] = 'tls://smtp.company.com:587';

$config['smtp_user'] = '%u';

$config['smtp_pass'] = '%p';

$config['product_name'] = 'Roundcube Webmail';

// You have to change this not leave it on default (24 char)
$config['des_key'] = 'pwa0gITVla1+mVaFi+66abJ6';

$config['plugins'] = [
    'archive',
    'zipdownload',
];

$config['skin'] = 'elastic';
```

After this settings, you can use your webmail service!