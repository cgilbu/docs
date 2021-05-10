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
`ufw allow 3306/tcp` Port 3306 (database)\
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

## Install HTTPS

`apt install python3-pip libaugeas0` Installs "pip" package manager plus a required module for certbot-apache\
`pip3 install certbot certbot-apache` Installs certificate provider with "Let's Encrypt"

## Install MySQL

`apt install mysql-server`\
`systemctl status mysql`

`mysql_secure_installation` Creates root user

`nano /etc/mysql/mysql.conf.d/mysqld.cnf`

```
bind-address = 0.0.0.0 # Enables remote connections
```

`systemctl restart mysql`

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

## Create database with user

`sudo mysql`

`CREATE DATABASE database;`\
`SHOW DATABASES;`

`CREATE USER 'chris'@'%' IDENTIFIED BY 'password';`\
`GRANT ALL PRIVILEGES ON database.* TO 'chris'@'%';`\
`FLUSH PRIVILEGES;`

`exit`

# Enable WebSockets

`sudo nano /etc/apache2/sites-available/domain.com-ssl.conf`

```
ProxyPass "/ws/" "ws://127.0.0.1:8080/"
ProxyPassReverse "/ws/" "ws://127.0.0.1:8080/"
```

`sudo a2enmod proxy proxy_http proxy_wstunnel`\
`sudo systemctl restart apache2`

## Run WebSocket

`sudo screen` Lets you run code on the server in the background\
`php /var/www/domain.com/public_html/your-sync-script.php` Runs your WebSocket sync script\
`CTRL AD` Detaches from screen

`sudo screen -r` Reconnects to screen\
`CTRL AK` Kills screen
