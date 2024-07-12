---
title: Sistema de encriptado de imágenes por áreas
description: La aplicación permite encriptar localmente secciones de imágenes que se almacenan en un servidor.
date: 2023-10-27
tags: [Python, Criptografía, Seguridad, Imágenes]
categories: [Proyectos, UC3M]
pin: false
toc: true
mermaid: true
image: 
    path: /media/2023-10-27-Sistema de encriptado de imagenes por areas/portada.webp
    alt: Portada del proyecto
---

## Sobre el proyecto

El objetivo de este proyecto es aprender conceptos de criptografía, seguridad y encriptado de datos. Para ello, se ha desarrollado una aplicación que permite encriptar secciones de imágenes que se almacenan en un servidor. Un caso de uso de esta aplicación sería la publicación de imágenes de cámaras de seguridad en tiempo real, donde se quiera proteger la privacidad de las personas que aparecen en las imágenes a la vez que se quiere poder visualizar la imagen original en caso de ser necesario.

## Funcionamiento

Al inicio de la aplicación, esta se intenta conectar al servidor, comprueba la validez y la confianza de su certificado y, si todo es correcto continua con la ejecución normal, donde el usuario puede ver imágenes **encriptadas** que el servidor almacena o iniciar sesión para ver sus propias imágenes desencriptadas. En realidad la verificación de certificados se realiza cada vez que se realiza una petición al servidor, por lo que si el certificado cambia, la aplicación lo detectará y mostrará un mensaje de error.
![](/media/2023-10-27-Sistema%20de%20encriptado%20de%20imagenes%20por%20areas/image4.png){: .white-bg}

Cabe destacar que el sistema no es realmente distribuido, ya que el servidor y el cliente están en la misma máquina y se representan como dos clases diferentes de la misma aplicación. Esto es solo una simplificación para poder mostrar el funcionamiento del sistema.

Si se quisiera desplegar el sistema en un entorno real, se debería separar el servidor y el cliente en dos máquinas diferentes y se debería implementar un sistema de comunicación entre ellos, como por ejemplo una API REST.

### Inicio de sesión y gestión de cuentas de usuario

#### Registro

1. Se conecta al servidor y se comprueba la validez y confianza del certificado.
2. Usuario introduce nombre de usuario y contraseña.
3. Se convierte la contraseña a binario.
4. Se encripta la contraseña la clave pública del servidor.
5. El servidor comprueba si el usuario ya existe.
6. Si no existe, desencripta la contraseña con su clave privada y comprueba que es valida.
7. Si es válida, se genera un HASH (KDF con Scrypt) de la contraseña y se almacena en la base de datos. De esta forma el servidor no almacena las contraseñas de los usuarios.
8. Se envía un mensaje de confirmación al usuario.

#### Inicio de sesión

1. Se conecta al servidor y se comprueba la validez y confianza del certificado.
2. Usuario introduce nombre de usuario y contraseña.
3. Se convierte la contraseña a binario.
4. Se encripta la contraseña la clave pública del servidor.
5. El servidor comprueba si el usuario existe.
6. Si existe, desencripta la contraseña con su clave privada.
7. Se genera el HASH (KDF con Scrypt) de la contraseña y se compara con el almacenado en la base de datos.
8. Si es válida, se envía un mensaje de confirmación al usuario.

### Encriptación y envío de imágenes
![](/media/2023-10-27-Sistema%20de%20encriptado%20de%20imagenes%20por%20areas/image1.jpg){: .white-bg}

1. Se carga la imagen en la aplicación y se seleccionan el rango de pixeles a encriptar.
2. Se comprueba el certificado del servidor.
3. Se encripta la sección de la imagen. 
    ![Desktop View](/media/2023-10-27-Sistema%20de%20encriptado%20de%20imagenes%20por%20areas/image3.jpg){: .white-bg  .h100}
   1. Se genera un salt aleatorio.
   2. Se genera una clave de encriptación con el salt y la contraseña (NO EL HASH) del usuario con PBKDF2HMAC.
   3. Se genera un IV aleatorio.
   4. Se escribe en el metadata de la imagen el IV, salt, algoritmo empleado y la sección de la imagen encriptada.
   5. Se encripta la sección de la imagen con AES en modo CTR..
4. Se firma la imagen con la clave privada del cliente.
   1. se genera una clave aleatoria.
   2. se genera un hash con SHA256 del binario de la imagen, el IV, el salt y la clave.
   3. Se firma (encripta) el hash con la clave privada del cliente.
   4. Se encripta la clave con la que se ha generado el hash con la clave pública del servidor.
   5. Se escribe en el metadata de la imagen el hash firmado y la clave encriptada.
5. Se envía la imagen al servidor.
   1. El servidor comprueba el certificado del cliente asi como sus credenciales.
   2. Se desencripta la clave con la clave privada del servidor.
   3. Regenera el hash con SHA256 del binario de la imagen, el IV, el salt y la clave.
   4. Se comprueba la firma del hash con la clave pública del cliente.
   5. Se almacena la imagen. 

**De esta manera el servidor no tiene acceso a la imagen original ni a la contraseña del usuario, por lo que no puede desencriptar la imagen.**

### Descarga y desencriptación de imágenes

1. Se conecta al servidor y se comprueba la validez y confianza del certificado.
2. Se solicita al servidor la/s imagen/es a desencriptar. Como estas son publicas se envían sin comprobación de credenciales.
3. Se desencripta cada imagen:
   1. Se leen de la metadata el IV, salt y la sección de la imagen encriptada.
   2. Se regenera la clave de encriptación con el salt y la contraseña del usuario.
   3. Se desencripta la sección de la imagen con AES en modo CTR.
   4. Se reconstruye la imagen original.
4. Se muestra la imagen al usuario.


## Estrucutra del proyecto

![](/media/2023-10-27-Sistema%20de%20encriptado%20de%20imagenes%20por%20areas/estructura.png){: .white-bg}


## Enlaces de interés

### Memoria del proyecto

La primera parte de la memoria del proyecto se puede encontrar en el siguiente enlace: [1º memoria del proyecto](/media/2023-10-27-Sistema%20de%20encriptado%20de%20imagenes%20por%20areas/Memoria 1.pdf)

La segunda parte de la memoria del proyecto se puede encontrar en el siguiente enlace: [2º memoria del proyecto](/media/2023-10-27-Sistema%20de%20encriptado%20de%20imagenes%20por%20areas/Memoria 2.pdf)

### Repositorio del proyecto

El código fuente de la ultima versión del proyecto se puede encontrar en el siguiente enlace: [Repositorio del proyecto](https://github.com/Ragarr/Criptografia_2023-24)