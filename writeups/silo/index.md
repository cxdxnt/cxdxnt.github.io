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

![GitHub Logo](/img/silo/enumeracion-sid.png)

Una vez obtenido el SID probamos a intentar iniciar session con contraseñas por default
