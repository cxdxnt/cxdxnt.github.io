## Maquina hackthebox Silo
![GitHub Logo](/img/silo/logo.png)

### Detalles
Silo es una maquina de hackthebox con una dificultal media en la intrusion y en la escalada de privilegios.Cuenta con un puerto
que corre el servicio Oracle.Oracle es un sistema de gestión de base de datos de tipo objeto-relacional, desarrollado por Oracle
Corporation.Mediante tecnicas informaticas logramos acceder a la maquina.Una ves dentro la escala a administrator se basa en un
volcado de memoria que mediante la herramienta volatility 3 logramos observar que hay adentro, una ves viendo procedemos a extraer
hashes , que gracias a esos hashes iniciamos session como administrador.

### Enumeracion 

Lanzamos nmap para enumerar los puertos y sus servicios que corren.Luego de que terminara el escaneo , veo que hay un servidor 1521 que corre el servicio oracle, me llamo bastante la atencion por su (unathorized), entonces fui directo a internet a buscar informacion de aquel puerto.
![GitHub Logo](/img/silo/nmap.PNG)

Buscando en internet encontre que para autenticarse con la base de datos necesitamos los sid.Los SID son  una instancia de base de datos única
Mediante hydra intentamos encontrar sus SID

![GitHub Logo](/img/silo/enumeracion-sid.PNG)

Una vez obtenido el SID probamos a intentar iniciar session con contraseñas por default , en este caso scott y tiger son el usuario y contraseña correctos

![GitHub Logo](/img/silo/connect oracle.PNG)

```markdown
as sysdba --> Es para poder entrar con un usuario con mas alto privilegios 

```

Una vez dentro nos creamos un usuario llamado cxdxnt 
![GitHub Logo](/img/silo/creando y elevando privilegios.PNG)
```markdown
CREATE USER  cxdxnt  IDENTIFIED BY cxdxnt#; ---> Crea un usuario
GRANT CONNECT , RESOURCE ,DBA TO cxdxnt --->  CONNECT para que el usuario pueda conectarse.RESOURCE (permitite al usuario crear tipos con nombre para esquemas personalizados)
                                              DBA  le permite al usuario no solo crear tipos con nombre personalizados, sino también modificarlos y destruirlos.
```

### ATAQUE:

Una vez creado al usuario , con la herramienta odat subimos una web shell 
![GitHub Logo](/img/silo/subiendo la web shell.PNG)
## Web shell
![GitHub Logo](/img/silo/mostrando la web shell.PNG)

Una vez subida la web shell procedemos a traer el backdoor
![GitHub Logo](/img/silo/trayendo la web shell a content.PNG)

Despues de a ver traido el backdoor desde la web shell , lo descargamos.

![GitHub Logo](/img/silo/descargando y ejecutando la reverse shell.PNG)

Y exito ya estamos dentro de la maquina :D

![GitHub Logo](/img/silo/acceso a la maquina.PNG)
### Elevacion de privilegios
Una vez dentro podemos observa la flag de user

![GitHub Logo](/img/silo/flag de user.PNG)

Ahora lo que queda es elevar a administrator.Observamos desde la web shell el archivo oracle issue.txt

![GitHub Logo](/img/silo/lo que dice el archivo oracle txt.PNG)

Procedemos a ingresar a la pagina y insertar la contraseña y observamos que hay un archivo .dmp , que seria un volcado de memoria 
![GitHub Logo](/img/silo/volcado de memoria.PNG)

## Forense 
Para ver el siguiente volcado de memoria usaremos volatility 3.Volatility3 es una herramienta forense de código abierto para la respuesta a incidentes y el análisis de malware. Dispone además de una serie de algoritmos rápidos y eficientes que le permiten analizar los volcados de memoria RAM.

Procedemos a ver lo que hay dentro : 

![GitHub Logo](/img/silo/analizando volcado de memoria.PNG)

Una vez viendo que hay archivos procedemos a ver si algun hash 

![GitHub Logo](/img/silo/buscando hash.PNG)

Y vamos resulta que hay un hash para ingresar como administrator a la maquina.

##Evil-winrm

Mediante cuya herramienta procedemos a conectarnos 

![GitHub Logo](/img/silo/connectarse a administrador y mostrar la flag.PNG)

Y listo ya tenemos acceso como administrator :)

## Script para automatizar la intrusion.
Estando aburrido me he creado un script, solo ejecutandolo ya te de acceso a la maquina :) .Para que funcione tienes que tener un par de cosas en cuenta :) con tal solo hecharle un ojo.

![GitHub Logo](/img/silo/script automatizado para acceder a la maquina.PNG)


link de script en python3 [GitHub](https://raw.githubusercontent.com/cxdxnt/script/main/new_silo_attack.py)





