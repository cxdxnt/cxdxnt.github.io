---
layout: single
title: Traverxec - Hack The Box
excerpt: "Traverxec era una caja relativamente fácil que implicaba enumerar y explotar un servidor web Nostromo. Aprovecharémos una vulnerabilidad RCE para obtener una shell en el host. Luego mediante una pala configuracion accedemos al user.Para obtener root, explotaré sudo usado con journalctrl."
date: 2022-09-27
classes: wide
header:
  teaser: /assets/images/htb-writeup-traverxec/traverxec_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - nostromo exploitation
  - abusing nostromo homedirs configuration
  - exploiting journalctl(privilege escalation)
---
![](/assets/images/htb-writeup-traverxec/traverxec_logo.png)

## Reconocimiento

```ruby
# Nmap 7.92 scan initiated Tue Sep 27 20:39:17 2022 as: nmap -sCV -p22,80 -oN targeted 10.10.10.165
Nmap scan report for 10.10.10.165
Host is up (0.24s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Sep 27 20:39:33 2022 -- 1 IP address (1 host up) scanned in 16.28 seconds
```

Vemos que el servidor web esta corriendo el servicio nostromo, entonces buscamos un exploit con searchsploit y vemos uno que nos permite ejecucion de codigo remoto.Procedemos a ejecutarlo.

![](/assets/images/htb-writeup-traverxec/exploit.png)


Aca podemos ver el proceso que hace el script en python para lograr ejecutar comandos.

![](/assets/images/htb-writeup-traverxec/data.png)

Una vez dentro del sistema vemos lo siguiente.

```bash

www-data@traverxec:/dev/shm$ cat /var/nostromo/conf/.htpasswd
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
www-data@traverxec:/dev/shm$ 

```
Procedemos a romper el hash con jhon

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:06 1,74% (ETA: 21:34:13) 0g/s 48478p/s 48478c/s 48478C/s southpoint..slideshow123
0g 0:00:00:25 7,47% (ETA: 21:34:02) 0g/s 48464p/s 48464c/s 48464C/s tinwater572..tinkey677
0g 0:00:00:28 8,58% (ETA: 21:33:54) 0g/s 49199p/s 49199c/s 49199C/s pooper55..pookey6
0g 0:00:01:43 37,95% (ETA: 21:32:59) 0g/s 53302p/s 53302c/s 53302C/s mibebetati..mibbbb
0g 0:00:01:47 39,43% (ETA: 21:32:59) 0g/s 53186p/s 53186c/s 53186C/s marinegrl..marinamiyarai
0g 0:00:01:48 39,92% (ETA: 21:32:58) 0g/s 53259p/s 53259c/s 53259C/s mamfa6..mameaw519112234
Nowonly4me       (?)
1g 0:00:02:50 DONE (2022-09-27 21:31) 0.005866g/s 62061p/s 62061c/s 62061C/s Noyoudo..Nous4=5
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
Pero dichas credenciales no nos sirve para nada.Procedemos a buscar que es nostromo y como podemos aprovecharnos de el.

```
es un servidor web de código abierto, también conocido como servidor web Nostromo, diseñado por Marcus Glocker. Se ejecuta como un proceso único y maneja conexiones concurrentes normales mediante llamadas seleccionadas, pero para lograr eficiencia durante conexiones más exigentes, como listados de directorios y ejecución de CGI, se bifurca.
```

Una vez leido dicho parrafo vemos que podemos ver los directorios del usuario david, entonces nos fijamos en el archivo nhttpd.conf

![](/assets/images/htb-writeup-traverxec/configuracion.png)

Extraemos una id_rsa y procedemos a crackearla.

![](/assets/images/htb-writeup-traverxec/id-rsa.png)

Una vez como el usuario david, vemos que puede ejecutar journalctl como root.Buscamos en gtfobins como podemos lograr una shell.Procedemos a minimizar la ventana hasta que quede de tal manera.

![](/assets/images/htb-writeup-traverxec/ejecucion.png)

Logramos ejecutar comandos como root.

![](/assets/images/htb-writeup-traverxec/root2.png)
