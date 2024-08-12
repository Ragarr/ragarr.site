---
title: Nextcloud en raid con ssl y dominio
description: Instalación de Nextcloud en un servidor con Debian 12, montaje de un raid 5 para almacenar los datos de forma segura y configuración del acceso desde el exterior de forma segura con un certificado SSL.
date: 2024-08-09
tags: [Nextcloud, Debian, Servidor, Apache, Raid, SSL, DDNS, Selfhosting]
categories: [Proyectos, Tutoriales]
pin: false
toc: true
comments: true
---

## Introduccion
Este tutorial tiene como objetivo instalar Nextcloud desde cero en un servidor con Debian 12, montar un raid 5 para almacenar los datos de forma segura y configurar el acceso desde el exterior de forma segura con un certificado SSL.
Cubriremos los siguientes puntos:
- Instalación y configuración de Debian 12
- Montaje de un raid 5 asegurar la integridad de los datos almacenados
- Configuración de un servidor web con Apache y PHP
- Instalación de Nextcloud
- Configuración de un DDNS para acceder desde el exterior
- Configuración de un certificado SSL con Let's Encrypt para asegurar la conexión desde el exterior
- Configuración de un firewall para proteger el servidor
- Configuración de un sistema de caché para mejorar el rendimiento de Nextcloud

Este tipo de configuración es ideal para reemplazar servicios como Google Drive, Dropbox o OneDrive, ya que nos permite tener un control total sobre nuestros datos y asegurar su integridad y privacidad.

### Requisitos
- Un ordenador viejo que vamos a usar como servidor que idealmente tenga:
  -  4GB de RAM o más (mínimo 2gb + 1gb por cada usuario activo).
  -  4 núcleos de CPU o más(mínimo recomendado 2).
  -  1 disco duro para el sistema operativo (mínimo 15-20gb).
  -  3 discos duros para el raid 5, tamaño a elección, que sean de la misma capacidad o similar, puedes calcular el tamaño ideal [aquí](https://www.seagate.com/es/es/products/nas-drives/raid-calculator/).
-  Acceso a la configuración del router.
-  Otro ordenador desde el que acceder y configurar el servidor.
## 1. Instalar debian 12

### 1.1. Descargar la imagen de debian 12

Descargar la imagen de debian 12 netinstall desde la [página oficial](https://www.debian.org/distrib/netinst)

### 1.2. Instalar debian 12
Grabar la imagen en un USB con [balenaEtcher](https://www.balena.io/etcher/) o [Rufus](https://rufus.ie/)

### Instalación
Instalamos el sistema operativo con las opciones que prefiramos, en un disco distinto del que vamos a usar para el raid 5.

Por ultimo, debemos desmarcar la opción de instalar el entorno gráfico. Unicamente instalaremos el sistema base y el servidor SSH.


Una vez finalizada la instalación, reiniciamos el sistema y accedemos con el usuario y la contraseña que hayamos establecido.

## 2. Montar raid 5

### 2.1. Instalar mdadm

```bash
apt install mdadm
```

### 2.2. Crear raid 5

Listar los discos
```bash	
fdisk -l
```
Miramos cuales son los discos que queremos usar para el raid 5, en este caso sda, sdb y sdc.º

```bash
mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sda /dev/sdb /dev/sdc
```

Verificar que se ha creado correctamente
```bash
sudo mdadm --detail /dev/md0
```

Crear un sistema de archivos
```bash
mkfs.ext4 /dev/md0
```

### 2.3. Montaje del raid 5
Montar el raid
```bash
sudo mkdir -p /mnt/raid
sudo mount /dev/md0 /mnt/raid
```

Configurar el montaje automático
```bash
sudo blkid /dev/md0
```
Nos dará un UUID, lo copiamos y editamos el archivo /etc/fstab
```bash
sudo nano /etc/fstab
```
Añadimos la siguiente línea
```bash
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /mnt/raid ext4 defaults 0 0
```

Guardar la configuración del raid
```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
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
```md
> Nota: Si no existe el directorio .ssh en el servidor, lo creamos
{:.info}
```

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

## 4. Instalar Nextcloud

### 4.1 Apache, MariaDB y PHP
#### 4.1.1 Instalar apache y mariadb
```bash
sudo apt update
sudo apt install apache2 mariadb-server
```
Hacemos una instalación segura de mariadb
```bash
sudo mariadb-secure-installation
```
A las preguntas respondemos
```bash
Enter current password for root (enter for none): <root_password>
Switch to unix_socket authentication [Y/n] n
change the root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```
#### 4.1.2 Preparar la base de datos
```bash
sudo mariadb -u root
```
```sql
create database nextcloud;
create user 'nextcloud'@'localhost' identified by '<nextcloud_password>';
grant all privileges on nextcloud.* to 'nextcloud'@'localhost';
flush privileges;
quit
```

#### 4.1.3 Instalar PHP y módulos necesarios
```bash
sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt upgrade
sudo apt install php8.2
sudo apt-get install -y php8.2 libapache2-mod-php8.2 libmagickcore-6.q16-6-extra php8.2-mysql php8.2-common php8.2-bz2 php8.2-redis php8.2-dom php8.2-curl php8.2-exif php8.2-fileinfo php8.2-bcmath php8.2-gmp php8.2-imagick php8.2-mbstring php8.2-xml php8.2-zip php8.2-iconv php8.2-intl php8.2-simplexml php8.2-xmlreader php8.2-ftp php8.2-ssh2 php8.2-sockets php8.2-gd php8.2-imap php8.2-soap php8.2-xmlrpc php8.2-apcu php8.2-dev php8.2-cli
```

### 4.2 Instalar Nextcloud
#### 4.2.1 Descargar Nextcloud
- Ve a la página de [Nextcloud](https://nextcloud.com/install/#instructions-server)
- Click en **COMMUNITY PROJECTS**
- En la seccion de **Archive** click derecho en **Get zip file** y copia la dirección del enlace
- Descarga el archivo en el servidor
```bash
wget https://download.nextcloud.com/server/releases/latest.zip
```

Descomprime el archivo
```bash
unzip latest.zip -d /var/www
```
```md	
> Nota: Si no tienes instalado unzip, instálalo con `sudo apt install unzip`
{:.info}
```



Cambia los permisos
```bash
cd /var/www
sudo chown -R www-data:www-data nextcloud/
```

#### 4.2.2 Configurar Apache
```bash
sudo a2enmod headers env dir mime rewrite
sudo service apache2 restart
systemctl restart apache2 # en caso de que no funcione el anterior
```

Edita el archivo de configuración de apache
```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```
Y añade las siguientes líneas
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

Guarda y reinicia apache
```bash
sudo service apache2 restart
sudo systemctl restart apache2 # en caso de que no funcione el anterior
```

#### 4.2.3 Configurar Nextcloud
##### 4.2.3.1 Configuración del router
Primero recomiendo asignar una dirección IP estática al servidor, esto es posible hacerlo desde la configuración del router. Cada router es diferente, por lo que no puedo dar una guía exacta, pero en la mayoría de los casos se puede hacer desde la sección de DHCP.

##### 4.2.3.2 Configuración de Nextcloud
Accede a la dirección IP del servidor desde un navegador y sigue los pasos de la instalación de Nextcloud.

- Establece el usuario y la contraseña administrador
- Establece la carpeta de datos
    - Carpeta de datos: /mnt/raid/nextcloud/data
- Establece la base de datos
    - Cuenta de base de datos: nextcloud
    - Contraseña de la base de datos: la que hayas establecido
    - Nombre de la base de datos: nextcloud
    - Servidor: localhost


```md
> Nota, si no has creado la carpeta de datos, hazlo con `sudo mkdir -p /mnt/raid/nextcloud/data` y asegúrate de que el usuario www-data tiene permisos de escritura en la carpeta con el comando `sudo chown -R www-data:www-data /mnt/raid/nextcloud/data`
{:.info}
```

Una vez termine de cargar le decimos de instalar las aplicaciones recomendadas.

Cuando el proceso termine, podemos solucionar algunos errores que nos aparecerán en la sección de administración.

### 4.3 Solucionar errores
#### 4.3.1 The PHP memory limit is below the recommended value of 512MB.
Edita el archivo de configuración de PHP
```bash
sudo nano /etc/php/8.2/apache2/php.ini
```
Y cambia la línea
```bash
memory_limit = 128M
```
Por
```bash
memory_limit = 512M
```

#### 4.3.2 Set up Cronjob for Nextcloud
```bash
sudo crontab -u www-data -e
```
Y añade la siguiente línea
```bash
*/5 * * * * php -f /var/www/nextcloud/cron.php
```

#### 4.3.3 No default phone region set
```bash	
sudo nano /var/www/nextcloud/config/config.php
```
Y añade la siguiente línea con el código de tu país
```php
'default_phone_region' => 'ES',
```

#### 4.3.4 No memory cache has been configured. 
Es muy recomendable configurar un sistema de caché, para ello vamos a usar Redis, que ya fue instalado durante el proceso de instalación de PHP.

Podemos compronbar que Redis está funcionando con el siguiente comando
```bash
ps ax | grep redis
```

Si no obtenemos un resultado similar a
```bash
  2488834 ?        Ssl    1:24 /usr/bin/redis-server 127.0.0.1:6379
```
Podemos instalar Redis con
```bash
sudo apt install redis-server php8.2-redis
```

Añadimos el usuario www-data al grupo redis y reiniciamos el servicio
```bash
sudo usermod -a -G redis www-data
sudo service apache2 restart
```

Editamos el archivo de configuración de Nextcloud
```bash
sudo nano /var/www/nextcloud/config/config.php
```

Añadimos las siguientes líneas
```php
    'filelocking.enabled' => true,
    'memcache.local' => '\\OC\\Memcache\\APCu',
    'memcache.locking' => '\\OC\\Memcache\\Redis',
    'redis' =>
    array (
        'host' => '127.0.0.1',
        'port' => 6379,
    ),
```

```md	
> Nota: Si en un futuro tienes errores con la subida y edicion de archivos y obtienes errores como "file is locked" o similares, prueba a desactivar el bloqueo de archivos con la siguiente línea `'filelocking.enabled' => false,`
{:.info}
```

Guardamos y reiniciamos apache
```bash
sudo service apache2 restart
```

#### 4.3.5 Fix php-imagick warning.
```bash	
sudo apt remove imagemagick-6-common php-imagick
sudo apt autoremove
sudo apt install imagemagick php-imagick
```	

#### 4.3.6 Fix the PHP module 'apcu' is not available.
```bash
sudo -u www-data php --define apc.enable_cli=1 /var/www/nextcloud/occ maintenance:repair
```

#### 4.3.7 El servidor no tiene una ventana de mantenimiento configurada
```bash
sudo nano /var/www/nextcloud/config/config.php
```
Y añade la siguiente línea
```php
  'maintenance_window_start' => '4',
```
Esto hara que la ventana de mantenimiento sea de 4 a 5 de la mañana.

## 5. Acceso seguro desde el exterior

### 5.1 Configurar un DDNS 
### 5.1.1 Crear un dominio en DynDNS
1. Crear una cuenta en [DynDNS](https://www.dyndns.com/account/create-account/)
2. En el menu DDNS, Dale a sign up
   1. Option 1: Use Our Domain Name
   2. Da un nombre a tu dominio
3. Guarda el dominio 

Añade una tarea a cron para que actualice la IP
```bash
crontab -e
```

Añade una linea como la siguiente
```bash
*/15 * * * * wget -O dynulog -4 "https://api.dynu.com/nic/update?hostname=example.dynu.com&myip=10.0.0.0&myipv6=no&username=someusername&password=somepassword"
```

Añade el dominio a la configuración de Nextcloud
```bash
sudo nano /var/www/nextcloud/config/config.php
```

Y añade la siguiente línea
```php
  'trusted_domains' =>
  array (
    0 => 'xxx.xxx.xxx.xxx',
    1 => 'tu_dominio.com',
    2 => 'subdominio.tu_dominio.com',
  ),
```

Guarda y reinicia apache
```bash
sudo service apache2 restart
```

### 5.2 Configurar un puerto en el router
Una vez has configurado el DDNS, debes configurar el router para permitir el acceso desde el exterior.

1. Accede a la configuración del router
2. Busca la sección de reenvío de puertos
3. Añade una nueva regla
   1. Nombre: Nextcloud
   2. Protocolo: TCP
   3. Puerto externo: 443
   4. IP interna: la IP del servidor
   5. Puerto interno: 443
   6. Guarda la configuración
4. Añade una segunda regla
   1. Nombre: Nextcloud
   2. Protocolo: TCP
   3. Puerto externo: 80
   4. IP interna: la IP del servidor
   5. Puerto interno: 80
   6. Guarda la configuración
5. Opcionalmente, puedes añadir una regla para el puerto 2222 (o el que eligieras) para acceder por ssh desde el exterior


```md


### 5.3 SSL (Let's Encrypt) 
Para poder acceder de forma segura a nuestro servidor desde el exterior, necesitamos un certificado SSL. Para ello vamos a usar Let's Encrypt.

#### 5.3.1 Instalar certbot
```bash
sudo apt install snapd
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

#### 5.3.2 Obtener un certificado
Este comando realizara la configuracion necesaria en apache y obtendra un certificado SSL de forma automatica. 
Tambien configurara la renovacion automatica del certificado.
```bash
sudo certbot --apache
```


Si has seguido todo correctamente tu archivo /etc/apache2/sites-available/000-default.conf debería tener una configuración similar a esta
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

Comprobamos que el certificado se auto-renueva correctamente
```bash
sudo certbot renew --dry-run
```

y si todo esta correcto deberiamos ver un mensaje similar a
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

Y con el comando ``systemctl list-timers`` deberiamos ver la siguiente linea
```bash
Tue 2024-07-23 00:03:00 CEST 9h left  -  -  snap.certbot.renew.timer  snap.certbot.renew.service
```
Que indica que se compronbará la renovación del certificado periodicamente.


## 6. Conclusiones
Con este tutorial hemos instalado Nextcloud en un servidor con Debian 12, montado un raid 5 para almacenar los datos de forma segura y hemos configurado el acceso desde el exterior de forma segura con un certificado SSL. 

Por lo que ahora podemos acceder a nuestros archivos de forma segura desde cualquier lugar y tener la tranquilidad de que nuestros datos están seguros.
