## Welcome to cxdxnt page 

![GitHub Logo](/img/nibbles/nibbles.png)

### Detalles

El dia de hoy le voy a mostrar la maquina nibbles de hackthebox, es una maquina linux de 64 bits que posee un nivel de dificulta facil en la intrusion y facil en la 
escalada de privilegios, la maquina posee una vulnerabilidad que nos permite la ejecucion de comandos(RCE) , cuando estamos en la maquinas y la enumeramos , nos damos 
cuenta que tiene una vulnerabilidad CVE-2017-1000112 , que nos permite a traves de una explotacion del kernel acceder a root 

### Enumeracion 
Lo primero que hacemos cuando queremos atacar una maquina es enumerarla entonces agarramos la poderosa herramienta nmap y procedemos a enumerarla:D

![GitHub Logo](/img/nibbles/nmap.PNG)

```markdown
-p  --> Puerto donde lanzamos los script
-oN --> Archivo en donde se guardan la salida de datos
-sC --> Realizar análisis con los scripts por defecto
-sV --> Detecta la version y servicio que esta corriendo
```
Vemos que tiene el puerto 80 , el puerto 80 esta como referido a que corre un sitio web.

Entramos a la pagina y le hechamos un vistazo al codigo fuente y contramos que tiene un directorio nibbleblog

![GitHub Logo](/img/nibbles/codigo_fuente.PNG)

Ahora que encontramos un directorio voy a usar la herramienta gobuster.

![GitHub Logo](/img/nibbles/gobuster.PNG)

```
dir --> Para decirle que queremos hacer un fuzzing a directorios de la web
-w --> Wordlist a utilizar(Diccionario de palabras que queremos que busque)
-u --> Web a atacar
-t --> Hilos(Le decimos a que velocidad deseamos que busque directorios)
-r --> Nos aplica un dereccionamiento
-x --> extenciones

```
Vemos que nos muestra mucho archivos pero lo que mas nos interesan son admin.php, ingresando en el directorio admin.php , nos damos cuenta que es un login
![GitHub Logo](/img/nibbles/admin-login.PNG)
Cuando veo un panel login lo que siempre hago es ingresar contraseñas por defecto como 
```
admin admin 
admin password
admin nibbles(el nombre de la maquina)
```
En este caso nos damos cuenta que admin, nibbles son el usuario y la contraseña correcta y procedemos a entrar y buscar en el panel.
### ATAQUE
Ahora si es el momento en que se tensa , buscando y explorando cada sitio de la pagina nos damos cuenta que podemos subir archivos , buscando en internet nos encontramos
un articulo que dice que la subida de archivo no es desinfectada entonces entramos a
![GitHub Logo](/img/nibbles/my_image.PNG)

Y procedemos a hacernos un script en php para la ejecucion de comandos 
![GitHub Logo](/img/nibbles/web_shell.PNG)

En el blog que nos encontramos anterior mente nos dice donde se guardan los archivos entonces procedemos a buscarlo

![GitHub Logo](/img/nibbles/donde se guarda la imagen.PNG)
Y pum la encontramos :D

Procedemos  a probar si tenemos ejecucion de comandos 

![GitHub Logo](/img/nibbles/rce.PNG)
Y bien logramos tener acceso , ahora procedemos a insertar una reverse shell por bash
![GitHub Logo](/img/nibbles/reverse shell.PNG)

Y sorpresa pudimos entrar a la computadora
![GitHub Logo](/img/nibbles/Connect.PNG)

### Elevacion de privilegios 
Una vez adentro empezamos a enumerar el sistema operativo, tiramos un comando que nos enseña la version del kernel 
![GitHub Logo](/img/nibbles/kernel.PNG)

Buscando por internet nos damos cuenta que su version de kernel es vulnerable y la descargamos , y con python nos abrimos un servidor para pasar el archivo
![GitHub Logo](/img/nibbles/server_python.PNG)

Desde la maquina victima procedemos a descargarlo
![GitHub Logo](/img/nibbles/Descargas.PNG)

Compilamos y ejecutamos el script y pum sorpresa :p
![GitHub Logo](/img/nibbles/root.PNG)

Ya elevamos privilegio a root :)


### Despedida

Gracias por visitar mi pagina si les interesa el tema y quieren aprender pueden preguntarme sin miedo :) adios
