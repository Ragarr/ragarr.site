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
  path: /media/2023-10-27-Sistema%20de%20encriptado%20de%20imagenes%20por%20areas/portada.webp
  alt: Project cover image
---

## About the project

The goal of this project was to learn key concepts of **cryptography, security, and data encryption**. We developed an application that allows encrypting **specific sections of images** before they are stored on a server. A typical use case would be real‑time publication of security camera feeds where privacy must be protected (e.g., blurring people’s faces) while still allowing access to the original image if necessary.

## How it works

When the application starts, it connects to the server, validates the server’s certificate, and, if trusted, continues execution. The user can then view encrypted images stored by the server or log in to view their own decrypted images. Certificate validation occurs on every request, so if the certificate changes, the application detects it and raises an error.

> Note: In this prototype the server and client run on the same machine as separate classes for demonstration purposes. In a real deployment, they would run on separate hosts and communicate over a channel such as a REST API.

### User account management

#### Registration
1. Connect to the server and validate its certificate.
2. User enters username and password.
3. Password is converted to binary.
4. Password is encrypted with the server’s public key.
5. Server checks if the user already exists.
6. If not, it decrypts the password with its private key and verifies it.
7. If valid, a password hash is generated (KDF with **Scrypt**) and stored in the database—so the server never stores raw passwords.
8. Confirmation is sent to the client.

#### Login
1. Connect to the server and validate its certificate.
2. User enters credentials.
3. Password is converted to binary.
4. Password is encrypted with the server’s public key.
5. Server checks if the user exists.
6. If so, it decrypts the password.
7. Generates the hash (KDF with **Scrypt**) and compares it with the stored value.
8. If valid, confirmation is sent to the client.

### Image encryption & upload

1. User loads an image and selects a pixel region to encrypt.
2. Server certificate is validated.
3. The selected region is encrypted using **AES‑CTR** with the following steps:  
   - Generate a random salt.  
   - Derive an encryption key with **PBKDF2HMAC** using the salt and the user’s password.  
   - Generate a random IV.  
   - Store IV, salt, algorithm, and region metadata in the image header.  
   - Encrypt the region with AES‑CTR.
4. The image is signed with the client’s private key:  
   - Generate a random key.  
   - Create a SHA256 hash of the image binary + IV + salt + key.  
   - Encrypt the hash with the client’s private key.  
   - Encrypt the random key with the server’s public key.  
   - Write the signed hash and encrypted key into the image metadata.  
5. Image is sent to the server:  
   - Server validates the client’s certificate and credentials.  
   - Decrypts the key with its private key.  
   - Recomputes the SHA256 hash.  
   - Verifies the signature with the client’s public key.  
   - Stores the image.

**This design ensures the server has no access to either the original image or the user’s password, so it cannot decrypt the content.**

### Image download & decryption

1. Client connects and validates the server’s certificate.
2. Requests images to decrypt. Since these are public, no credentials are required.
3. For each image:  
   - Read IV, salt, and encrypted region from metadata.  
   - Re‑derive the key using salt + user’s password.  
   - Decrypt the region with AES‑CTR.  
   - Reconstruct the full original image.
4. Decrypted image is displayed to the user.

## Project structure

![](/media/2023-10-27-Sistema%20de%20encriptado%20de%20imagenes%20por%20areas/estructura.png){: .white-bg}

## References

### Project reports
- [Project Report – Part 1](/media/2023-10-27-Sistema%20de%20encriptado%20de%20imagenes%20por%20areas/Memoria%201.pdf)  
- [Project Report – Part 2](/media/2023-10-27-Sistema%20de%20encriptado%20de%20imagenes%20por%20areas/Memoria%202.pdf)

### Source code
- [GitHub repository](https://github.com/Ragarr/Criptografia_2023-24)