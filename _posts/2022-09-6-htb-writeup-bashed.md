---
layout: single
title: bashed - Hack The Box
excerpt: "Bashed es una máquina sencilla que cuenta con una webshell expuesta en la web, luego de ganar acceso a la máquina vemos que el usuario www-data tiene permiso de ejecutar comandos como scriptmanager. Luego para la escala de privilegios a root vemos que el usuario root está ejecutando el archivo test.py"
date: 2022-09-6
classes: wide
header:
  teaser: /assets/images/htb-writeup-bashed/bashed_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - webshell 
  - script
  - crontab
---
![](/assets/images/htb-writeup-bashed/bashed_logo.png)

Bashed es una máquina sencilla que cuenta con una webshell expuesta en la web, luego de ganar acceso a la máquina vemos que el usuario www-data tiene permiso de ejecutar comandos como scriptmanager. Luego para la escala de privilegios a root vemos que el usuario root está ejecutando el archivo test.py, procedemos a modificarlo y ganar acceso como el usuario root.

## Enumeración 

Lanzamos un simple escaneo de nmap y encontramos que tiene solamente el puerto 80 abierto.
```ruby
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-07 08:45 -03
Nmap scan report for 10.10.10.68
Host is up (0.24s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Arrexel's Development Site
|_http-server-header: Apache/2.4.18 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.80 seconds
```

Navegamos por la web, y vemos que en la ruta “http://10.10.10.68/dev/phpbash.php” está subida una web shell.

![](/assets/images/htb-writeup-bashed/shell.png)

## Escalada de privilegios
La escalada de privilegios es bastante fácil, como el usuario www-data ejecutamos ``` sudo -l ``` y vemos que tiene permisos de ejecutar comandos como el usuario scriptmanager.


```bash
www-data@bashed:/tmp$ sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
www-data@bashed:/tmp$ sudo -u  scriptmanager bash
scriptmanager@bashed:/tmp$ 
```
Ahora toca escalar a root :). Con el pspy vemos que a intervalos de tiempo root ejecuta una tarea cron. Lo que hace es simplemente ejecutar el script test.py

```bash
2022/09/07 05:56:01 CMD: UID=0    PID=15383  | python test.py 
2022/09/07 05:56:01 CMD: UID=0    PID=15382  | /bin/sh -c cd /scripts; for f in *.py; do python "$f"; done 
2022/09/07 05:57:01 CMD: UID=0    PID=15386  | python test.py 
2022/09/07 05:57:01 CMD: UID=0    PID=15385  | /bin/sh -c cd /scripts; for f in *.py; do python "$f"; done 
2022/09/07 05:57:01 CMD: UID=0    PID=15384  | /usr/sbin/CRON -f 
2022/09/07 05:57:26 CMD: UID=0    PID=15388  | 
2022/09/07 05:57:26 CMD: UID=0    PID=15387  | /bin/systemd-tmpfiles --clean 
2022/09/07 05:57:26 CMD: UID=0    PID=15389  | 
2022/09/07 05:57:26 CMD: UID=0    PID=15390  | 
2022/09/07 05:58:01 CMD: UID=0    PID=15392  | /usr/sbin/CRON -f 
2022/09/07 05:58:01 CMD: UID=0    PID=15391  | /usr/sbin/CRON -f 
2022/09/07 05:58:01 CMD: UID=0    PID=15393  | python test.py 
```
Modificamos el archivo test.py y esperamos :).

![](/assets/images/htb-writeup-bashed/root.png)

Lo logramos :D.
