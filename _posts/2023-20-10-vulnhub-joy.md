---
layout: single
title: JOY - VulnHub
excerpt: "\"JOY\" es una máquina vulnerable disponible en VulnHub que presenta una vulnerabilidad de fuga de información en el servidor FTP. Esta vulnerabilidad nos permite utilizar el protocolo TFTP para acceder al contenido de archivos, lo que, a su vez, nos brinda la oportunidad de identificar las versiones del software involucrado. Es importante destacar que la versión ProFTPD 1.3.5 se encuentra vulnerable a una técnica de subida de archivos que aprovecharemos para cargar una webshell. Posteriormente, la escalación de privilegios se basa en la exposición de una contraseña y una configuración incorrecta de los scripts de bash."
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

"JOY" es una máquina vulnerable disponible en VulnHub que presenta una vulnerabilidad de fuga de información en el servidor FTP. Esta vulnerabilidad nos permite utilizar el protocolo TFTP para acceder al contenido de archivos, lo que, a su vez, nos brinda la oportunidad de identificar las versiones del software involucrado. Es importante destacar que la versión ProFTPD 1.3.5 se encuentra vulnerable a una técnica de subida de archivos que aprovecharemos para cargar una webshell. Posteriormente, la escalación de privilegios se basa en la exposición de una contraseña y una configuración incorrecta de los scripts de bash.

## Enumeracion

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios.

```bash
# Nmap 7.93 scan initiated Mon Oct 16 16:30:15 2023 as: nmap -sCV -p21,22,25,80,110,139,143,445,465,587,993,995 -oN scanvuln.nmap 192.168.56.9
Nmap scan report for 192.168.56.9
Host is up (0.00028s latency).

PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         ProFTPD 1.2.10
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxr-x   2 ftp      ftp          4096 Jan  6  2019 download
|_drwxrwxr-x   2 ftp      ftp          4096 Jan 10  2019 upload
22/tcp  open  ssh         Dropbear sshd 0.34 (protocol 2.0)
25/tcp  open  smtp        Postfix smtpd
| ssl-cert: Subject: commonName=JOY
| Subject Alternative Name: DNS:JOY
| Not valid before: 2018-12-23T14:29:24
|_Not valid after:  2028-12-20T14:29:24
|_ssl-date: TLS randomness does not represent time
|_smtp-commands: JOY.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8
80/tcp  open  http        Apache httpd 2.4.25
| http-ls: Volume /
| SIZE  TIME              FILENAME
| -     2016-07-19 20:03  ossec/
|_
|_http-title: Index of /
|_http-server-header: Apache/2.4.25 (Debian)
110/tcp open  pop3        Dovecot pop3d
|_pop3-capabilities: PIPELINING UIDL RESP-CODES STLS CAPA AUTH-RESP-CODE SASL TOP
|_ssl-date: TLS randomness does not represent time
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
|_imap-capabilities: IDLE ENABLE more LOGINDISABLEDA0001 LOGIN-REFERRALS STARTTLS listed SASL-IR capabilities ID IMAP4rev1 post-login OK LITERAL+ Pre-login have
|_ssl-date: TLS randomness does not represent time
445/tcp open  netbios-ssn Samba smbd 4.5.12-Debian (workgroup: WORKGROUP)
465/tcp open  smtp        Postfix smtpd
|_smtp-commands: JOY.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=JOY
| Subject Alternative Name: DNS:JOY
| Not valid before: 2018-12-23T14:29:24
|_Not valid after:  2028-12-20T14:29:24
587/tcp open  smtp        Postfix smtpd
|_smtp-commands: JOY.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=JOY
| Subject Alternative Name: DNS:JOY
| Not valid before: 2018-12-23T14:29:24
|_Not valid after:  2028-12-20T14:29:24
993/tcp open  ssl/imaps?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=JOY/organizationName=Good Tech Pte. Ltd/stateOrProvinceName=Singapore/countryName=SG
| Not valid before: 2019-01-27T17:23:23
|_Not valid after:  2032-10-05T17:23:23
995/tcp open  ssl/pop3s?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=JOY/organizationName=Good Tech Pte. Ltd/stateOrProvinceName=Singapore/countryName=SG
| Not valid before: 2019-01-27T17:23:23
|_Not valid after:  2032-10-05T17:23:23
MAC Address: 08:00:27:82:1D:79 (Oracle VirtualBox virtual NIC)
Service Info: Hosts: The,  JOY.localdomain, 127.0.1.1, JOY; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: JOY, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2023-10-16T16:30:26
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.5.12-Debian)
|   Computer name: joy
|   NetBIOS computer name: JOY\x00
|   Domain name: \x00
|   FQDN: joy
|_  System time: 2023-10-17T00:30:26+08:00
|_clock-skew: mean: -5h40m01s, deviation: 4h37m07s, median: -3h00m02s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Oct 16 16:30:58 2023 -- 1 IP address (1 host up) scanned in 43.26 seconds
```


## ¿Por dónde empezar?
General mente cuando hay una gran cantidad  de puertos, comienzo primero por los servicios que no tienen un servidor web. Acá primero comienzo por(ftp,smbmap) y luego por los servidores web.


### FTP

En este servicio, hemos descubierto información que resultará útil más adelante.

![](/assets/images/vulnhub-writeup-joy/ftp_joy.png)

Dentro del archivo "directory", observamos una salida que parece ser el resultado de un comando "ls -la"

```bash
➜  ftp cat directory    
Patrick's Directory

total 108
drwxr-xr-x 18 patrick patrick 4096 Oct 17 00:30 .
drwxr-xr-x  4 root    root    4096 Jan  6  2019 ..
-rw-r--r--  1 patrick patrick    0 Oct 17 00:30 1gNbzYGjQJGBGRUOGnWVIXNgLVanG5l7.txt
-rw-------  1 patrick patrick  185 Jan 28  2019 .bash_history
-rw-r--r--  1 patrick patrick  220 Dec 23  2018 .bash_logout
-rw-r--r--  1 patrick patrick 3526 Dec 23  2018 .bashrc
drwx------  7 patrick patrick 4096 Jan 10  2019 .cache
drwx------ 10 patrick patrick 4096 Dec 26  2018 .config
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Desktop
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Documents
drwxr-xr-x  3 patrick patrick 4096 Jan  6  2019 Downloads
drwx------  3 patrick patrick 4096 Dec 26  2018 .gnupg
-rwxrwxrwx  1 patrick patrick    0 Jan  9  2019 haha
-rw-------  1 patrick patrick 8532 Jan 28  2019 .ICEauthority
drwxr-xr-x  3 patrick patrick 4096 Dec 26  2018 .local
drwx------  5 patrick patrick 4096 Dec 28  2018 .mozilla
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Music
drwxr-xr-x  2 patrick patrick 4096 Jan  8  2019 .nano
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Pictures
-rw-r--r--  1 patrick patrick  675 Dec 23  2018 .profile
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Public
-rw-r--r--  1 patrick patrick   24 Oct 17 00:30 rTZ5tbESiyVd2iILaE44R1rGwvNIhqxwg1mZliv0uqJGeRopXMtFrvdWXu7nYZEz.txt
d---------  2 root    root    4096 Jan  9  2019 script
drwx------  2 patrick patrick 4096 Dec 26  2018 .ssh
-rw-r--r--  1 patrick patrick    0 Jan  6  2019 Sun
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Templates
-rw-r--r--  1 patrick patrick    0 Jan  6  2019 .txt
-rw-r--r--  1 patrick patrick  407 Jan 27  2019 version_control
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Videos

You should know where the directory can be accessed.

Information of this Machine!

Linux JOY 4.9.0-8-amd64 #1 SMP Debian 4.9.130-2 (2018-10-27) x86_64 GNU/Linux
```

### SMB(SAMBA)

En este servicio, no tenemos acceso a ninguna carpeta compartida, lo que significa que nuestras opciones para realizar acciones adicionales están limitadas.


### HTTP(Servidor Web)

En este servicio, solo disponemos de un directorio que no contiene información valiosa.

![](/assets/images/vulnhub-writeup-joy/ossec.png)


## Escaneo UDP

Dado que no hemos logrado progresar a través de los servicios y puertos identificados en el protocolo TCP, consideremos ahora la posibilidad de realizar un análisis a través del protocolo UDP.

```bash
# Nmap 7.93 scan initiated Wed Oct 18 08:16:28 2023 as: nmap -sU --top-port 600 --open -v -n -Pn -T5 -oN udp-scan.nmap 192.168.56.9
Warning: 192.168.56.9 giving up on port because retransmission cap hit (2).
Nmap scan report for 192.168.56.9
Host is up (0.0033s latency).
Not shown: 582 open|filtered udp ports (no-response), 15 closed udp ports (port-unreach)
PORT    STATE SERVICE
123/udp open  ntp
137/udp open  netbios-ns
161/udp open  snmp
MAC Address: 08:00:27:82:1D:79 (Oracle VirtualBox virtual NIC)

Read data files from: /usr/bin/../share/nmap
# Nmap done at Wed Oct 18 08:16:38 2023 -- 1 IP address (1 host up) scanned in 9.84 seconds
```

Hemos encontrado algo de interés en el puerto SNMP, por lo tanto, procedemos a investigarlo. La información relevante se ha guardado en el archivo "snmp.txt".

```bash
snmpbulkwalk -c public -v2c 192.168.56.9 > snmp.txt
```

Hemos logrado identificar una pista sumamente interesante que arroja luz sobre el contenido del archivo "directory".

![](/assets/images/vulnhub-writeup-joy/snmp.png)

Hemos realizado una verificación para determinar si el puerto está abierto.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-18 08:24 -03
Nmap scan report for 192.168.56.9
Host is up (0.00029s latency).

PORT      STATE         SERVICE
36969/udp open|filtered unknown
MAC Address: 08:00:27:82:1D:79 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.38 seconds
```

### ¿Qué es el servició TFTP?

>Hacktricks nos dice que :
>**TFTP** utiliza el puerto UDP 69 y **no requiere autenticación** - los clientes leen y escriben en los servidores utilizando el formato de datagrama descrito en RFC 1350. Debido a las deficiencias del protocolo (es decir, la falta de autenticación y la falta de seguridad en el transporte), es poco común encontrar servidores en Internet público. Sin embargo, en grandes redes internas, TFTP se utiliza para servir archivos de configuración e imágenes ROM a teléfonos VoIP y otros dispositivos.
>Pero en este caso vimmos que camban el puerto a 36969. Ahora procedemos a conectarnos.


![](/assets/images/vulnhub-writeup-joy/tftp.png)

Ahora, vamos a considerar cómo podemos aprovechar esta información. Previamente, nos proporcionaron la lista de archivos en el directorio /home/patrick, por lo tanto, podemos utilizar tftp para listar su contenido de la siguiente manera.

```bash
ftp> ?
Commands may be abbreviated.  Commands are:

connect 	connect to remote tftp
mode    	set file transfer mode
put     	send file
get     	receive file
quit    	exit tftp
verbose 	toggle verbose mode
trace   	toggle packet tracing
status  	show current status
binary  	set mode to octet
ascii   	set mode to netascii
rexmt   	set per-packet retransmission timeout
timeout 	set total retransmission timeout
?       	print help information
tftp> get .bashrc
Received 3639 bytes in 0.0 seconds
```

Hemos descargado el archivo ".bashrc" que aparece en la salida del archivo "directory".

![](/assets/images/vulnhub-writeup-joy/bashr.png)

Una vez confirmado que tenemos la capacidad de descargar archivos, procedemos a obtener un archivo llamado "version_control".


![](/assets/images/vulnhub-writeup-joy/versiones.png)

Hemos realizado una búsqueda minuciosa de las versiones y hemos identificado que el servicio ProFTPd es vulnerable a una subida de archivos. Aprovecharemos esta vulnerabilidad para cargar una webshell. La ubicación de los archivos web está detallada en el archivo "version_control".

![](/assets/images/vulnhub-writeup-joy/exploit-joy.png)

Si analizamos lo que realiza el exploit para llevar a cabo esta tarea de manera manual, el proceso se desglosa de la siguiente manera:


![](/assets/images/vulnhub-writeup-joy/analisis.png)

Si analizamos el tráfico con Wireshark, observaremos la siguiente información:

![](/assets/images/vulnhub-writeup-joy/wirehark.png)

Al parecer, el proceso no inicia una sesión, sino que únicamente ejecuta comandos.

![](/assets/images/vulnhub-writeup-joy/exploit-manual.png)

Logramos visualizar el archivo subido.

![](/assets/images/vulnhub-writeup-joy/reverse-shell.png)

## Escalación de privilegios

Al acceder a la máquina comprometida, hemos encontrado el directorio "ossec" y un archivo que proporciona credenciales.

![](/assets/images/vulnhub-writeup-joy/privilege.png)


### Inicio de sessión

```bash
www-data@JOY:/var/www/tryingharderisjoy/ossec$ cat patricksecretsofjoy 
credentials for JOY:
patrick:apollo098765
root:howtheheckdoiknowwhattherootpasswordis

how would these hack3rs ever find such a page?
```

Hemos observado que tenemos la capacidad de ejecutar el script "test" como usuario root.

```bash
patrick@JOY:/root$ sudo -l
Matching Defaults entries for patrick on JOY:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User patrick may run the following commands on JOY:
    (ALL) NOPASSWD: /home/patrick/script/test
```

El script parece requerir que elijamos un archivo y luego le otorguemos permisos, lo que plantea una seria preocupación en términos de seguridad, ya que esto podría abrir numerosas vulnerabilidades que podrían utilizarse para escalar privilegios.

![](/assets/images/vulnhub-writeup-joy/sudo-root.png)

En consecuencia, hemos llevado a cabo la asignación del permiso Set-UID (SUID) al ejecutable "/bin/bash", lo que ha posibilitado la obtención de acceso con privilegios de root.

![](/assets/images/vulnhub-writeup-joy/real_pwned.png)
