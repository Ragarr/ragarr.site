---
title: Image Encryption System by Areas
description: Application to locally encrypt selected regions of images before storing them on a server.
date: 2023-10-27
tags: [Python, Cryptography, Security, Images]
categories: [Projects, UC3M]
pin: false
toc: true
mermaid: true
lang: en
image:
  path: /media/2023-10-27-Image encryption system/portada.webp
  alt: Project cover image
---

## About the project

The objective of this project is to learn concepts of cryptography, security, and data encryption. To this end, an application has been developed that allows sections of images stored on a server to be encrypted. One use case for this application would be the publication of real-time security camera images, where the privacy of the people appearing in the images needs to be protected, while at the same time being able to view the original image if necessary.

## How it works

When the application starts, it attempts to connect to the server, checks the validity and trustworthiness of its certificate, and, if everything is correct, continues with normal execution, where the user can view **encrypted** images stored by the server or log in to view their own decrypted images. In reality, certificate verification is performed each time a request is made to the server, so if the certificate changes, the application will detect it and display an error message.
![](/media/2023-10-27-Image encryption system/image4.png){: .white-bg}

It should be noted that the system is not truly distributed, since the server and client are on the same machine and are represented as two different classes of the same application. This is only a simplification to show how the system works.

If you wanted to deploy the system in a real environment, you would have to separate the server and client onto two different machines and implement a communication system between them, such as a REST API.

### Login and user account management

#### Registration

1. Connect to the server and check the validity and trustworthiness of the certificate.
2. User enters username and password.
3. Convert the password to binary.
4. Encrypt the password with the server's public key.
5. The server checks if the user already exists.
6. If not, it decrypts the password with its private key and checks that it is valid.
7. If valid, a HASH (KDF with Scrypt) of the password is generated and stored in the database. This way, the server does not store user passwords.
8. A confirmation message is sent to the user.

#### Login

1. Connect to the server and check the validity and trustworthiness of the certificate.
2. User enters username and password.
3. Convert the password to binary.
4. Encrypt the password with the server's public key.
5. The server checks if the user exists.
6. If they exist, the password is decrypted with their private key.
7. The password's HASH (KDF with Scrypt) is generated and compared with the one stored in the database.
8. If it is valid, a confirmation message is sent to the user.

### Image encryption and sending
![](/media/2023-10-27-Image encryption system/image1.jpg){: .white-bg}

1. The image is loaded into the application and the range of pixels to be encrypted is selected.
2. The server certificate is checked.
3. The section of the image is encrypted.
    ![Desktop View](/media/2023-10-27-Image encryption system/image3.jpg){: .white-bg  .h100}
   1. A random salt is generated.
   2. An encryption key is generated with the salt and the user's password (NOT THE HASH) using PBKDF2HMAC.
   3. A random IV is generated.
   4. The IV, salt, algorithm used, and encrypted image section are written to the image metadata.
   5. The section of the image is encrypted with AES in CTR mode.
4. The image is signed with the client's private key.
   1. A random key is generated.
   2. A hash is generated with SHA256 from the image binary, the IV, the salt, and the key.
   3. The hash is signed (encrypted) with the client's private key.
   4. The key used to generate the hash is encrypted with the server's public key.
   5. The signed hash and encrypted key are written to the image metadata.
5. The image is sent to the server.
   1. The server checks the client's certificate and credentials.
   2. The key is decrypted with the server's private key.
   3. The hash is regenerated with SHA256 from the image binary, the IV, the salt, and the key.
   4. The hash signature is verified with the client's public key.
   5. The image is stored. 

**This way, the server does not have access to the original image or the user's password, so it cannot decrypt the image.**

### Downloading and decrypting images

1. Connect to the server and check the validity and trustworthiness of the certificate.
2. Request the image(s) to be decrypted from the server. As these are public, they are sent without credential verification.
3. Each image is decrypted:
   1. The IV, salt, and section of the encrypted image are read from the metadata.
   2. The encryption key is regenerated with the salt and the user's password.
   3. The section of the image is decrypted with AES in CTR mode.
   4. The original image is reconstructed.
4. The image is displayed to the user.

## Project structure

![](/media/2023-10-27-Image encryption system/estructura.png){: .white-bg}

## Links of interest
### Project report

The first part of the project report can be found at the following link: [1st project report](/media/2023-10-27-Image encryption system/Memoria 1.pdf)

The second part of the project report can be found at the following link: [2nd project report](/media/2023-10-27-Image encryption system/Memoria 2.pdf)

### Project repository

The source code for the latest version of the project can be found at the following link: [Project repository](https://github.com/Ragarr/Criptografia_2023-24)