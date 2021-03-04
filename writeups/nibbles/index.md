## Welcome to cxdxnt page 

![GitHub Logo](/img/nibbles/nibbles.png)

### Detalles

El dia de hoy le voy a mostrar la maquina nibbles de hackthebox, es una maquina linux de 64 bits que posee un nivel de dificulta facil en la intrusion y facil en la 
escalada de privilegios, la maquina posee una vulnerabilidad que nos permite la ejecucion de comandos(RCE) , cuando estamos en la maquinas y la enumeramos , nos damos 
cuenta que tiene una vulnerabilidad CVE-2017-1000112 , que nos permite a traves de una explotacion del kernel acceder a root 

### Enumeracion 
Lo primero que hacemos cuando queremos atacar una maquina es enumerarla entonces agarramos la poderosa herramienta nmap y procedemos a enumerarla :D
![GitHub Logo](/img/nibbles/nmap.PNG)

```markdown
-p  Los puertos que deseamos saber que corren 
-oN Archivo en donde queremos que guarde los datos
-sC realizar análisis con los scripts por defecto
-sV Detecta la version y servicio que esta corriendo
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/cxdxnt/cxdxnt.github.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.
