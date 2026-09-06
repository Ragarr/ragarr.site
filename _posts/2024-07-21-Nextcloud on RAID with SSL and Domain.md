---
title: Nextcloud on RAID with SSL and Domain (Archived Tutorial)
description: Archived tutorial on installing Nextcloud with Debian 12, RAID5, and SSL. Outdated practices, kept only for reference.
date: 2024-08-09
tags: [Nextcloud, Debian, Server, Apache, RAID, SSL, DDNS, Selfhosting, archived]
categories: [Systems, Homelab]
pin: false
toc: true
comments: true
---

> **Disclaimer:** This tutorial was written in mid‑2024 as one of my first self‑hosting experiments. Some practices are **outdated or insecure**. I keep this post online **only for historical/reference purposes**.

## Introduction

This tutorial aimed to install Nextcloud from scratch on a Debian 12 server, configure a RAID5 array for data storage, and secure external access with SSL certificates.

It covers:

* Debian 12 installation and setup
* RAID5 configuration
* Apache and PHP setup
* Nextcloud installation
* DDNS for external access
* SSL certificates via Let’s Encrypt
* Firewall and caching configuration

*(Content below remains as originally written in 2024, for archival purposes only.)*

---

## 1. Install Debian 12

### 1.1. Download the Debian 12 image

Download the Debian 12 netinstall image from the [official website](https://www.debian.org/distrib/netinst)

### 1.2. Install Debian 12
Burn the image to a USB drive using [balenaEtcher](https://www.balena.io/etcher/) or [Rufus](https://rufus.ie/)

### Installation
Install the operating system with your preferred options on a different disk from the one you are going to use for RAID 5.

Finally, uncheck the option to install the graphical environment. Only install the base system and the SSH server.


Once the installation is complete, restart the system and log in with the username and password you have set.

## 2. Mount RAID 5

### 2.1. Install mdadm

```bash
apt install mdadm
```

### 2.2. Create RAID 5

List the disks
```bash    
fdisk -l
```
We look at which disks we want to use for RAID 5, in this case sda, sdb, and sdc.

```bash
mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sda /dev/sdb /dev/sdc
```

Verify that it has been created correctly
```bash
sudo mdadm --detail /dev/md0
```

Create a file system
```bash
mkfs.ext4 /dev/md0
```

### 2.3. Mounting the RAID 5
Mount the RAID
```bash
sudo mkdir -p /mnt/raid
sudo mount /dev/md0 /mnt/raid
```

Configure automatic mounting
```bash
sudo blkid /dev/md0
```
This will give us a UUID. Copy it and edit the /etc/fstab file
```bash
sudo nano /etc/fstab
```
Add the following line
```bash
UUID=xxxx
```

## 3. Asegurar el servidor
### 3.1. Cambiar el puerto ssh
```bash
sudo nano /etc/ssh/sshd_config
```
Y encontraremos la línea
```bash
# Port 22
```
La descomentamos y cambiamos el puerto 22 por el que queramos, por ejemplo 2222
```bash
Port 2222
```
Guardamos y reiniciamos el servicio
```bash
sudo systemctl restart sshd
```

### 3.2 instalar UFW
```bash
sudo apt install ufw
```

Configuramos el firewall, sustituyendo el puerto 2222 por el que hayamos elegido
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH'
ufw allow http
ufw allow https
sudo ufw enable
```
Comprobamos que el firewall está activo
```bash
sudo ufw status
# ufw reload # para recargar la configuración en caso de cambios
```

### 3.3 Keygen para ssh
**En el cliente** ejecutamos el siguiente comando, sustituyendo `<client_name>` por el nombre elijamos.

```bash
ssh-keygen -t rsa -b 4096 -C "<client_name>"
```

Vamos al archivo donde se ha guardado la clave pública y mandamos la clave al servidor
```bash
cat ~/.ssh/id_rsa.pub
```

Manda la clave al servidor
```bash
scp -P 2222 ~/.ssh/id_rsa.pub <user>@<server_ip>:/home/<user>/.ssh/authorized_keys
```

> Nota: Si no existe el directorio .ssh en el servidor, lo creamos
{: .prompt-info }

**Desde el servidor** hacemos que solo se acepten conexiones ssh con clave
```bash
sudo nano /etc/ssh/sshd_config
```
Y cambiamos las siguientes líneas
```bash
#PasswordAuthentication yes
#PermitRootLogin yes
```
Por
```bash
PasswordAuthentication no
PermitRootLogin no
```

Reiniciamos el servicio
```bash
sudo systemctl restart sshd
```

Ahora cuando intentemos acceder al servidor por ssh, deberia localizar la clave privada y no pedirnos la contraseña.

## 4. Install Nextcloud

### 4.1 Apache, MariaDB, and PHP
#### 4.1.1 Install Apache and MariaDB
```bash
sudo apt update
sudo apt install apache2 mariadb-server
```
Perform a secure installation of MariaDB
```bash
sudo mariadb-secure-installation
```
We answer the questions
```bash
Enter current password for root (enter for none): <root_password>
Switch to unix_socket authentication [Y/n] n
change the root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```
#### 4.1.2 Prepare the database
```bash
sudo mariadb -u root
```
```sql
create database nextcloud;
create user ‘nextcloud’@'localhost' identified by ‘<nextcloud_password>’;
grant all privileges on nextcloud.* to ‘nextcloud’@'localhost';
flush privileges;
quit
```

#### 4.1.3 Install PHP and necessary modules
```bash
sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt upgrade
sudo apt install php8.2
sudo apt-get install -y php8.2 libapache2-mod-php8.2 libmagickcore-6.q16-6-extra php8.2-mysql php8. 2-common php8.2-bz2 php8.2-redis php8.2-dom php8.2-curl php8.2-exif php8.2-fileinfo php8.2-bcmath php8.2-gmp php8.2-imagick php8.2-mbstring php8.2-xml php8.2-zip php8. 2-iconv php8.2-intl php8.2-simplexml php8.2-xmlreader php8.2-ftp php8.2-ssh2 php8.2-sockets php8.2-gd php8.2-imap php8.2-soap php8.2-xmlrpc php8.2-apcu php8.2-dev php8.2-cli
```

### 4.2 Install Nextcloud
#### 4.2.1 Download Nextcloud
- Go to the [Nextcloud](https://nextcloud.com/install/#instructions-server) website
- Click on **COMMUNITY PROJECTS**
- In the **Archive** section, right-click on **Get zip file** and copy the link address
- Download the file to the server
```bash
wget https://download.nextcloud.com/server/releases/latest.zip
```

Unzip the file
```bash
unzip latest.zip -d /var/www
```

> Note: If you don't have unzip installed, install it with `sudo apt install unzip`
{: .prompt-info }



Change the permissions
```bash
cd /var/www
sudo chown -R www-data:www-data nextcloud/
```

#### 4.2.2 Configure Apache
```bash
sudo a2enmod headers env dir mime rewrite
sudo service apache2 restart
systemctl restart apache2 # if the previous command does not work
```

Edit the Apache configuration file
```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```
And add the following lines
```apache
<VirtualHost *:80>

    ServerName tu_dominio.com
    DocumentRoot /var/www/nextcloud

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews

        <IfModule mod_dav.c>
            Dav off
        </IfModule>

        RewriteEngine On
        RewriteRule ^/\.well-known/carddav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
        RewriteRule ^/\.well-known/caldav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
        RewriteRule ^/\.well-known/host-meta https://%{SERVER_NAME}/public.php?service=host-meta [QSA,L]
        RewriteRule ^/\.well-known/host-meta\.json https://%{SERVER_NAME}/public.php?service=host-meta-json [QSA,L]
        RewriteRule ^/\.well-known/webfinger https://%{SERVER_NAME}/public.php?service=webfinger [QSA,L]

    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

Save and restart Apache
```bash
sudo service apache2 restart
sudo systemctl restart apache2 # if the previous command does not work
```

#### 4.2.3 Configuring Nextcloud
##### 4.2.3.1 Router configuration
First, I recommend assigning a static IP address to the server. This can be done from the router settings. Each router is different, so I cannot provide exact instructions, but in most cases it can be done from the DHCP section.

##### 4.2.3.2 Nextcloud configuration
Access the server's IP address from a browser and follow the steps to install Nextcloud.

- Set the administrator username and password
- Set the data folder
- Data folder: /mnt/raid/nextcloud/data
- Set the database
    - Database account: nextcloud
- Database password: the one you have set
- Database name: nextcloud
- Server: localhost



> Note: if you have not created the data folder, do so with `sudo mkdir -p /mnt/raid/nextcloud/data` and make sure that the www-data user has write permissions for the folder with the command `sudo chown -R www-data:www-data /mnt/raid/nextcloud/data`
{: .prompt-info }

Once the upload is complete, we will tell you to install the recommended applications.

When the process is complete, we can fix some errors that will appear in the administration section.

### 4.3 Troubleshooting errors
#### 4.3.1 The PHP memory limit is below the recommended value of 512MB.
Edit the PHP configuration file
```bash
sudo nano /etc/php/8.2/apache2/php.ini
```
And change the line
```bash
memory_limit = 128M
```
To
```bash
memory_limit = 512M
```

#### 4.3.2 Set up Cronjob for Nextcloud
```bash
sudo crontab -u www-data -e
```
And add the following line
```bash
*/5 * * * * php -f /var/www/nextcloud/cron.php
```

#### 4.3.3 No default phone region set
```bash    
sudo nano /var/www/nextcloud/config/config.php
```
And add the following line with your country code
```php
‘default_phone_region’ => ‘ES’,
```

#### 4.3.4 No memory cache has been configured. 
It is highly recommended to configure a cache system. To do this, we will use Redis, which was already installed during the PHP installation process.

We can check that Redis is working with the following command
```bash
ps ax | grep redis
```

If we don't get a result similar to
```bash
  2488834 ?        Ssl    1:24 /usr/bin/redis-server 127.0.0.1:6379
```
We can install Redis with
```bash
sudo apt install redis-server php8.2-redis
```

Add the www-data user to the redis group and restart the service
```bash
sudo usermod -a -G redis www-data
sudo service apache2 restart
```

Edit the Nextcloud configuration file
```bash
sudo nano /var/www/nextcloud/config/config.php
```

Add the following lines
```php
    ‘filelocking.enabled’ => true,
    ‘memcache.local’ => ‘\\OC\\Memcache\\APCu’,
    ‘memcache.locking’ => ‘\\OC\\Memcache\\Redis’,
    ‘redis’ =>
    array (
        ‘host’ => ‘127.0.0.1’,
        ‘port’ => 6379,
    ),
```


> Note: If you encounter errors when uploading and editing files in the future and receive messages such as “file is locked” or similar, try disabling file locking with the following line: `‘filelocking.enabled’ => false,`
{: .prompt-info }

Save and restart Apache
```bash
sudo service apache2 restart
```

#### 4.3.5 Fix php-imagick warning.
```bash    
sudo apt remove imagemagick-6-common php-imagick
sudo apt autoremove
sudo apt install imagemagick php-imagick
```    

#### 4.3.6 Fix the PHP module ‘apcu’ is not available.
```bash
sudo -u www-data php --define apc.enable_cli=1 /var/www/nextcloud/occ maintenance:repair
```

#### 4.3.7 The server does not have a maintenance window configured
```bash
sudo nano /var/www/nextcloud/config/config.php
```
And add the following line
```php
  ‘maintenance_window_start’ => ‘4’,
```
This will set the maintenance window from 4 to 5 a.m.

## 5. Secure access from outside

### 5.1 Configure a DDNS
### 5.1.1 Create a domain in DynDNS
1. Create an account at [DynDNS](https://www.dyndns.com/account/create-account/)
2. In the DDNS menu, click sign up
   1. Option 1: Use Our Domain Name
   2. Give your domain a name
3. Save the domain

Add a task to cron to update the IP
```bash
crontab -e
```

Add a line like the following
```bash
*/15 * * * * wget -O dynulog -4 “https://api.dynu.com/nic/update?hostname=example.dynu.com&myip=10.0.0.0&myipv6=no&username=someusername&password=somepassword”
```

Add the domain to the Nextcloud configuration
```bash
sudo nano /var/www/nextcloud/config/config.php
```

And add the following line
```php
  ‘trusted_domains’ =>
  array (
    0 => ‘xxx.xxx.xxx.xxx’,
    1 => ‘your_domain.com’,
    2 => ‘subdomain.your_domain.com’,
  ),
```

Save and restart Apache
```bash
sudo service apache2 restart
```

### 5.2 Configure a port on the router
Once you have configured DDNS, you must configure the router to allow external access.

1. Access the router settings
2. Find the port forwarding section
3. Add a new rule
   1. Name: Nextcloud
   2. Protocol: TCP
   3. External port: 443
   4. Internal IP: the server's IP
   5. Internal port: 443
   6. Save the configuration
4. Add a second rule
   1. Name: Nextcloud
   2. Protocol: TCP
   3. External port: 80
   4. Internal IP: the server's IP address
   5. Internal port: 80
   6. Save the configuration
5. Optionally, you can add a rule for port 2222 (or whichever you chose) to access via SSH from outside.

### 5.3 SSL (Let's Encrypt) 
To securely access our server from outside, we need an SSL certificate. To do this, we will use Let's Encrypt.

#### 5.3.1 Install certbot
```bash
sudo apt install snapd
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

#### 5.3.2 Obtain a certificate
This command will perform the necessary configuration in Apache and automatically obtain an SSL certificate. 
It will also configure automatic certificate renewal.
```bash
sudo certbot --apache
```


If you have followed everything correctly, your /etc/apache2/sites-available/000-default.conf file should have a configuration similar to this
```apache
<VirtualHost *:80>

    ServerName your_domain.com
    DocumentRoot /var/www/nextcloud

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews

        <IfModule mod_dav.c>
            Dav off
        </IfModule>

        RewriteEngine On
        RewriteRule ^/\.well-known/carddav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
        RewriteRule ^/\.well-known/caldav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
        RewriteRule ^/\.well-known/host-meta https://%{SERVER_NAME}/public.php?service=host-meta [QSA,L]
        RewriteRule ^/\.well-known/host-meta\.json https://%{SERVER_NAME}/public.php?service=host-meta-json [QSA,L]
        RewriteRule ^/\.well-known/webfinger https://%{SERVER_NAME}/public.php?service=webfinger [QSA,L]

    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

RewriteCond %{SERVER_NAME} =your_domain.com [OR]
RewriteCond %{SERVER_NAME} =your_domain.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
```

We verify that the certificate renews itself correctly
```bash
sudo certbot renew --dry-run
```

and if everything is correct, we should see a message similar to
```bash
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/<your_domain>.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Account registered.
Simulating renewal of an existing certificate for <your_domain>

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations, all simulated renewals succeeded:
  /etc/letsencrypt/live/<your_domain>/fullchain.pem (success)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

And with the command ``systemctl list-timers``, we should see the following line
```bash
Tue 2024-07-23 00:03:00 CEST 9h left  -  -  snap.certbot.renew.timer  snap.certbot.renew.service
```
This indicates that the certificate renewal will be checked periodically.

## 6. Conclusions
In this tutorial, we have installed Nextcloud on a server running Debian 12, set up a RAID 5 array to store data securely, and configured secure external access with an SSL certificate.

Now we can securely access our files from anywhere and rest assured that our data is safe.
