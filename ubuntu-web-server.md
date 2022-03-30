# How to configure Apache web server on Ubuntu

This guide will help you configure an Apache web server with a website running on PHP and MySQL. It will also enable HTTP/2 and HTTPS and show you how to run WebSockets.

## Upgrade Ubuntu

`apt update` Gets available upgrades\
`apt upgrade`

## Configure Firewall

`apt install ufw` Installs "Uncomplicated Firewall"\
`ufw allow http` Port 80\
`ufw allow https` Port 443\
`ufw allow ssh` Port 22\
`ufw enable`\
`ufw status verbose`

## Install Apache

`apt install apache2`

## Install PHP with FastCGI and MySQL

`apt install php php7.*-fpm php-mysql` Installs PHP with FastCGI (FPM) and MySQL extensions\
`a2enmod proxy_fcgi setenvif` Enables FastCGI (FPM)\
`a2enconf php7.*-fpm` Enables FastCGI (FPM)\
`a2dismod php7.*` Disables old PHP module without FastCGI (FPM)\
`systemctl restart apache2`

## Enable HTTP/2

`a2dismod mpm_prefork` MPM prefork doesn't support HTTP/2\
`a2enmod mpm_event http2` Switches to MPM event and enables HTTP/2\
`systemctl restart apache2`

## Install SSL (HTTPS)

`apt install python3-pip libaugeas0` Installs "pip" package manager plus a required module for certbot-apache\
`pip3 install zope.interface --upgrade` Upgrades package required for certbot\
`pip3 install certbot certbot-apache` Installs certificate provider with "Let's Encrypt"

`nano /etc/systemd/system/certbot.service`

```
[Unit]
Description=Let's Encrypt Renewal

[Service]
Type=oneshot
ExecStart=certbot renew --quiet --agree-tos --renew-hook "systemctl reload apache2"
```

`nano /etc/systemd/system/certbot.timer`

```
[Unit]
Description=Daily Renewal of Let's Encrypt Certificates

[Timer]
OnCalendar=daily
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

`systemctl start certbot.timer`\
`systemctl enable certbot.timer` Enables automatic restart after server restart\
`systemctl status certbot.timer`

## Install MySQL

`apt install mysql-server`\
`systemctl status mysql`

`mysql_secure_installation` Creates root user

## Disable Password Authentication

`nano /etc/ssh/sshd_config`\
`PasswordAuthentication no`\
`service ssh restart`

## Create User

`adduser chris`\
`usermod -aG sudo chris` Adds user to sudo group\
`su - chris` Switches to user\
`mkdir ~/.ssh` Creates SSH folder\
`chmod 700 ~/.ssh` Grants user full folder access\
`nano ~/.ssh/authorized_keys` Add public key here

Exit and reconnect with new user

# Add New Domain

`sudo a2dissite 000-default.conf` Disables default website

`sudo nano /etc/hosts` Add domain name here\
`sudo mkdir -p /var/www/domain.com/public_html` Creates website folder

`sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/domain.com.conf`\
`sudo nano /etc/apache2/sites-available/domain.com.conf`

```
<VirtualHost>
  ServerAdmin your@email.com
  DocumentRoot /var/www/domain.com/public_html
  ServerName domain.com

  ErrorLog domain.com-error.log
  CustomLog domain.com-access.log
</VirtualHost>
```

`sudo a2ensite domain.com.conf`\
`sudo systemctl restart apache2`

`sudo certbot --apache -d domain.com` Enables HTTPS (works only after domain has been pointed)

Grant file access (see below)

## Grant File Access

`sudo usermod -aG www-data chris` Adds user to www-data group

`sudo chown -R :www-data /var/www` Makes www-data group owner of www-folder\
`sudo chmod -R 774 /var/www` Grants full folder access to owner and group and read access to others

## Disable Directory Listing (globally)

`sudo nano /etc/apache2/conf-available/apache2-tweaks.conf`

```
<Directory /var/www/>
  Options -Indexes
</Directory>
```

`sudo a2enconf apache2-tweaks.conf`\
`sudo systemctl reload apache2`

## Create database with user

`sudo mysql`

`CREATE DATABASE database;`\
`SHOW DATABASES;`

`CREATE USER 'chris'@'%' IDENTIFIED BY 'password';`\
`GRANT ALL PRIVILEGES ON database.* TO 'chris'@'%';`\
`FLUSH PRIVILEGES;`

`exit`

# Enable WebSockets

`sudo nano /etc/apache2/sites-available/domain.com-le-ssl.conf`

```
ProxyPass "/ws/" "ws://127.0.0.1:8080/"
ProxyPassReverse "/ws/" "ws://127.0.0.1:8080/"
```

`sudo a2enmod proxy proxy_http proxy_wstunnel`\
`sudo systemctl restart apache2`

## Run WebSocket

`sudo nano /etc/systemd/system/your-websocket-sync.service`

```
[Unit]
Description=Your Sync

[Service]
Type=simple
Restart=always
User=chris
ExecStart=/usr/bin/env php /var/www/domain.com/public_html/your-websocket-sync.php

[Install]
WantedBy=multi-user.target
```

`sudo systemctl start your-websocket-sync.service`\
`sudo systemctl enable your-websocket-sync.service` Enables automatic restart after server restart\
`sudo systemctl status your-websocket-sync.service`

# Configure CORS for Cross-Origin Requests

Necessary if domain.com is requesting stuff from other domains like api.domain.com. This should go in .htaccess (instead of the site config) if the CORS is project specific.

`sudo nano /etc/apache2/sites-available/domain.com-le-ssl.conf`

```
<Directory /var/www/domain.com/public_html>
  SetEnvIf Origin "^(https:\/\/domain.com|http:\/\/localhost)$" DOMAIN=$0
  Header always set Access-Control-Allow-Origin "%{DOMAIN}e"
  Header always set Access-Control-Allow-Methods "GET, POST, OPTIONS"
  Header always set Access-Control-Allow-Headers "Content-Type"

  RewriteEngine on
  RewriteCond %{REQUEST_METHOD} OPTIONS
  RewriteRule ^(.*)$ $1 [R=200,L]
</Directory>
```

`sudo systemctl reload apache2`

# Enable .htaccess

`sudo nano /etc/apache2/sites-available/domain.com-le-ssl.conf`

```
<Directory /var/www/choices.redcreek.no/public_html>
  Options FollowSymLinks
  AllowOverride all
  Require all granted
</Directory>
```
