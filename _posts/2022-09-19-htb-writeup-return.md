---
layout: single
title: Return - Hack The Box
excerpt: "La máquina return es una máquina de hackthebox de dificultad fácil que consiste en un abusing printer, que extraemos una contraseña que nos sirve para conectarnos a la máquina.Luego el usuario svc-printer,que extraemos una contraseña que nos sirve para conectarnos a la maquina.Luego el usuario svc-printer forma parte del grupo server operators los miembros de este grupo pueden iniciar y detener los servicios del sistema,entonces iniciamos un servicio que tengas una reverse shell para escalar a nt authority."
date: 2022-09-19
classes: wide
header:
  teaser: /assets/images/htb-writeup-return/return_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - cookie manipulation
  - abusing printer
  - server operrator
  - service configuracion manipulation
---

![](/assets/images/htb-writeup-return/return_logo.png)

## Enumeración

Aplicamos una enumeración con nmap.

```ruby
# Nmap 7.92 scan initiated Mon Sep 19 04:59:08 2022 as: nmap -sCV -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49674,49675,49679,49682,49694 -oN targeted 10.10.11.108
Nmap scan report for 10.10.11.108
Host is up (0.24s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: HTB Printer Admin Panel
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-09-19 13:18:02Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49679/tcp open  msrpc         Microsoft Windows RPC
49682/tcp open  msrpc         Microsoft Windows RPC
49694/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 5h18m44s
| smb2-time: 
|   date: 2022-09-19T13:19:01
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Sep 19 05:00:31 2022 -- 1 IP address (1 host up) scanned in 83.16 seconds
```

Cuando termina el escaneo de nmap, vemos que el servicio 80 http está abierto. Una vez que entramos a la web nos dirigimos a settings, y vemos 4 entradas de input, cambiamos el Server Address por nuestra ip, y nos ponemos en escucha en el puertop 389.


![](/assets/images/htb-writeup-return/validando.png)

## Elevación de privilegios

Una vez dentro como el usuario svc-printer, aplicamos un comando para ver en que grupos forma parte. Y vemos que forma parte del grupo server operators, lo que este grupo nos permite es configurar un servicio. Configuramos un servicio que tenga una reverse shell, y lo paramos e iniciamos devuelta para que los cambios sean aplicados.

![](/assets/images/htb-writeup-return/administrator.png)
