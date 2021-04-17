### Welcome to cxdxnt page 

![GitHub Logo](/img/forense/insiden/descripsion.PNG)

## Detalles 
Hoy nos pegamos un descanso de hacking y nos vamos al forense , este reto consiste en una amenaza , unos ciberdelicuentes entraron a una empresa y los CEOS quieren averiguar que informacion obtuvieron.Entonces mediante tecnicas forense vamos a ir a veriguarlo :). 

### Enumerando 

Una vez descomprimimos el archivo ,pasamos a buscar la carpeta Profile , ¿Porque la carpeta Profile? Basicamente por que ahi se almacenan las contraseñas y las busquedas que ha hecho el usuario.Logramos encontrarlo :D 

![GitHub Logo](/img/forense/insiden/buscar archivo profile.PNG)

### Buscamos password y history.

Una vez encontramos el archivo profile , nos centramos en el archivo 2542z9mo.default-release.Empezamos a fijarnos que se ha fijado dicho usuario malisioso.

![GitHub Logo](/img/forense/insiden/buscar history.PNG)

Vemos ha accedido a un panel de login.¿Como lo sabemos? En tomcat se usa la ruta manager para acceder a una cuenta , entonces damos por entender que ha robado credenciales.Ahora pasamos a ver que contraseñas se ha llevado dicho usuario. 

![GitHub Logo](/img/forense/insiden/buscando password.PNG)

Encontramos dos cadenas de texto cifradas, mediante la herramienta firefox_decryp , vamos a descifrarlasa :)

![GitHub Logo](/img/forense/insiden/decryp.PNG)

Y listo logramos ver la flag y hemos completado el reto :)

