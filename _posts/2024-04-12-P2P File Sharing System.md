---
title: P2P File Sharing System
description: Development of a P2P file sharing system with a logger using ONC-RPC and a C-based server (indexer), a Python Web Service, and a Python client.
date: 2024-04-12
tags: [distributed systems, c, python, onc-rpc, web service, p2p, sockets]
categories: [Coursework, Distributed Systems]
pin: false
toc: true
lang: en
image:
  path: /media/2024-04-12-P2P%20File%20Sharing%20System/portada.png
  alt: System architecture model
---

## About the project

This project was developed as the final assignment for the **Distributed Systems** course. The objective was to build a distributed system that allows users to **share files among peers**.

The system consists of several components:
- A **main server** acting as an indexer.  
- An **RPC server** (logger) that records user actions and maintains an auditable log external to the main server.  
- A **web service** providing current date and time.  
- A **client** that allows users to interact with the server and each other to share files.

The full project can be found in the following [GitHub repository](https://github.com/Ragarr/UC3M/tree/main/Proyectos%20y%20practicas/3%C2%BA/SSDD/Proyecto-SSDD/Proyecto-SSDD-main).

## System architecture

![](/media/2024-04-12-P2P%20File%20Sharing%20System/fig1.png)

### RPC Service (Logger)
Acts as a **central logging system**, capturing and storing all user operations. Implemented in **C** using the **ONC-RPC** model, it ensures consistent interfaces for requests/responses and provides effective auditing.

### Server
Handles **all interactions** within the system: registration, connections, and content publishing. Implemented in **C** as a **multithreaded concurrent server** using **TCP sockets**, it supports multiple concurrent client connections.

### Web Service
Provides the **current date and time** for clients to timestamp their operations. Implemented in **Python**, it can be deployed locally or remotely, returning data as strings for easy integration.

### Client
Enables users to issue commands to **interact with the server**, **publish files**, and **download files directly from peers**. Implemented in **Python**, it operates via a **command-line interpreter** and also manages direct P2P connections. In P2P mode it behaves as a **multithreaded server**, supporting multiple simultaneous file transfers.

## Message structure by component

Messages follow the **course specification**: all error codes are **8-bit integers**, while other data is sent as **strings** terminated by `\0`. Since communication is handled byte by byte, no endianness conversion (ntoh/hton) is required.

### RPC Service (Logger)
Messages are strings containing **username, executed operation, and timestamp**. The RPC server responds with a status code (0 = success, -1 = error).

### Server
Uses a **TCP-based protocol** with messages like `REGISTER, UNREGISTER, CONNECT, DISCONNECT, PUBLISH, DELETE`. Each request includes the **command, timestamp, and username**, plus additional parameters (e.g., filename). Responses are 8-bit status codes, sometimes followed by extra data (`LIST USERS`, `LIST CONTENT`).

### Web Service
Handles a single request: return **date and time** in string format via HTTP. Implemented with SOAP/XML for structured messaging.

### Client
Communicates with the server using the above **TCP-based protocol**. Commands include:  
`REGISTER, UNREGISTER, CONNECT, DISCONNECT, PUBLISH, DELETE, LIST_USERS, LIST_CONTENT`.  
Clients can also directly communicate via **P2P (TCP)** using the `GET_FILE` command. File transfers are performed byte-by-byte to ensure integrity.

## Application protocols

### Client–Server
Communication follows a **client-server, request-response model over TCP**:  
1. **Connection**: client opens a TCP connection.  
2. **Request**: sends command and arguments.  
3. **Processing**: server executes logic, interacts with RPC Logger if needed.  
4. **Response**: server sends status + optional data.  
5. **Disconnection**: connection closes (not persistent).  

This design saves resources but does not detect abrupt client disconnects (suggested improvement: implement heartbeats).

### Client–Client (P2P)
Peers connect via TCP to transfer files. Clients obtain peer IP/port from the central server.  
Steps: connection, request (file name), file transfer, confirmation, close.  
TCP ensures reliability to prevent file corruption.

### Server–RPC Logger
Communication uses **ONC-RPC**. The main server performs remote calls to log actions transparently, as if calling local functions.

### Client–Web Service
The web service exposes a **SOAP interface** over HTTP. The client requests date/time for timestamping operations.

## Compilation & execution

The project includes a `setup.py` script to automate compilation and execution.

Options:  
- `--clean|-c`: clean generated files.  
- `--build|-b`: compile server, RPC service, and install dependencies.  
- `--run|-r`: run RPC service, web server, and server.  

Typical usage:  
```bash
$ python3 setup.py -cbr
```
This performs a clean build and runs all services. Clients are then executed separately.

## Test suite

Unit tests were implemented using **Python’s unittest** and `unittest.mock` for simulating responses. Real processes were also launched for concurrency testing.

### `test_client_invalids.py`
Verifies client behavior under invalid inputs and server errors. Uses socket and web service mocks.

### `test_client_valids.py`
Valid client operations tested with mocks and controlled responses.

### `test_multiclient.py`
Tests concurrency with multiple clients and real server processes. Includes concurrent downloads and registrations.

### `test_server.py`
Covers server-side behavior with real processes. Includes registration, connection, publication, deletion, listing, and disconnection, plus failure cases.

## Additional considerations

Potential improvements:  
- Better client feedback (progress bars, clearer error messages).  
- Stronger security (access controls, encryption).  
- Server persistence (store state in files instead of memory).  
- Cleaner component modularization (avoid tight coupling during compilation).  
- Heartbeat mechanism for client liveness detection.

## Conclusion

This project was a **challenging and rewarding exercise** that put into practice nearly all concepts from the **Distributed Systems** course.

