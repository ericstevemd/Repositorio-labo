# PICKLE RICK

Máquina de nivel principiante de **TryHackMe**. Rick se ha convertido en pepinillo y no puede volver: hay que entrar en su ordenador y encontrar los **tres ingredientes** de la poción. Todo pasa por una web con un panel de comandos.

- **IP:** `10.64.145.255`
- **Servicios:** 22 (SSH) y 80 (web)

## 1. Reconocimiento

Como primer paso, realizamos el escaneo de la máquina con `nmap`:

```
nmap -sC -sV -p- 10.64.145.255
```

![[Pasted image 20261024195817.png]]

Quedan abiertos dos puertos:

- **22/tcp** → OpenSSH 8.2p1 (Ubuntu).
- **80/tcp** → Apache httpd 2.4.41 (página web), con el título **"Rick is sup4 cool"**.

El puerto 22 no parece el camino: SSH casi nunca es la primera puerta de entrada en estas máquinas. Seguimos por la web.

## 2. La página y el código fuente

Al abrir la IP en el navegador aparece la página "Help Morty!", donde Rick le pide a Morty que entre en su ordenador y encuentre los tres ingredientes:

![[Pasted image 20261024195858.png]]

No hay ningún formulario de login a la vista, así que miramos el código fuente de la página (Ctrl+U o inspeccionar elemento). Ahí hay un comentario HTML que no debería estar:

![[Pasted image 20261024195916.png]]

```html
<!--
    Note to self, remember username!
    Username: R1ckRul3s
-->
```

**Usuario encontrado:** `R1ckRul3s`.

Tenemos usuario pero no dónde usarlo: ni en SSH (solo acepta clave) ni en la propia página. Hay que buscar un formulario de login oculto.

## 3. Enumeración de directorios con Gobuster

Con el puerto 80 abierto, pasamos a enumerar rutas con `gobuster`. El detalle importante es la flag `-x`: sin extensiones solo busca nombres de carpetas, y aquí lo que nos interesa son los `.php` y `.txt`:

```
gobuster dir -u http://10.64.145.255 -w /home/kali/SecLists/Discovery/Web-Content/common.txt -x php,js,txt,old
```

![[Pasted image 20261024200734.png]]

Lo relevante de los resultados (el resto es ruido de `.ht*` con 403):

- `/assets` (301) → imágenes y estáticos de la página.
- `/login.php` (200) → ¡el formulario que buscábamos!
- `/portal.php` y `/denied.php` (302) → redirigen a `/login.php`.
- `/robots.txt` (200) → siempre merece una ojeada.
- `/server-status` (403) → sin permisos, descartado.

## 4. robots.txt: la contraseña

Abrimos `robots.txt` a mano en el navegador y en lugar de las típicas instrucciones para rastreadores nos encontramos esto:

![[Pasted image 20261024201634.png]]

```
Wubbalubbadubdub
```

Es el grito de guerra de Rick, pero en un `robots.txt` raro... **probamos que sea la contraseña.**

## 5. Login y Command Panel

En `/login.php` combinamos lo que hemos ido juntando:

- **Usuario:** `R1ckRul3s` (comentario HTML)
- **Contraseña:** `Wubbalubbadubdub` (`robots.txt`)

Entramos y aparece el **"Rick Portal"**. Todas las pestañas del menú (`Potions`, `Creatures`, `Beth Clone Notes`...) llevan a `denied.php` y están bloqueadas, así que lo único útil es **Commands**, un panel donde podemos ejecutar comandos Linux en el servidor:

![[Pasted image 20261024200911.png]]

## 6. Primer ingrediente desde el panel

Lo primero de siempre: ver dónde estamos y qué hay.

```
ls
```

![[Pasted image 20261024200931.png]]

```
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

El nombre del primer archivo se delata solo. Probamos a leerlo, pero **`cat` está bloqueado** por el panel (`Command disabled...`). No pasa nada: hay muchas formas de volcar un archivo sin usar `cat` → `less`, `tac`, `head`, `tail`, `grep`, `strings`...

![[Pasted image 20261024200955.png]]

> **Primer ingrediente: `mr. meeseek hair`**

`clue.txt` además nos avisa de que el resto está *"alrededor del sistema de archivos"*, no en la web.

## 7. Reverse shell

El panel sirve, pero es cómodo: se corta, no hay TTY y cada comando hay que esperar a que recargue. Para trabajar con calma montamos una reverse shell.

**Paso 1.** En nuestra máquina (Kali) abrimos el oyente:

```
nc -lvnp 4444
```

**Paso 2.** Preparamos el payload de perl, apuntando a **nuestra IP** (`192.168.131.171`) y el puerto `4444` del oyente. La captura guarda el one-liner tal cual:

![[Pasted image 20261024201555.png]]

Básicamente abre un socket TCP hacia nosotros, redirige las tres streams estándar (`STDIN`, `STDOUT`, `STDERR`) a ese socket y ejecuta una shell interactiva. Nada que no se haga en cualquier tutorial de reverse shells.

**Paso 3.** Lo pegamos en el Command Panel y damos a *Execute*. La conexión vuelve a nuestro `nc`:

![[Pasted image 20261024201759.png]]

Ya estamos dentro como `www-data`.

> **Nota sobre antivirus:** Defender marca el one-liner completo como `Backdoor:Perl/...`. Es esperable: es una reverse shell, su función es conectar y dar un shell. **El comando completo solo vive en la captura de arriba** (una imagen no la escanea nadie); si lo necesitas copiado y pegado, hazlo desde dentro de la VM de Kali y no lo guardes como archivo en Windows.

## 8. Terminal interactiva

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

## 9. Escalada a root

El usuario del servidor web (`www-data`) tiene permisos de `sudo` sin contraseña, así que directamente nos convertimos en root:

```
sudo su
```

Ya somos `root@ip-10-64-145-255` (en la captura de la shell se ve el cambio de prompt). Fuera del panel ya **no está bloqueado `cat`**, así que recogemos los tres ingredientes.

## 10. Los tres ingredientes

### 1er ingrediente

Estaba en la web, junto a los archivos del sitio:

```
cat Sup3rS3cretPickl3Ingred.txt
```

![[Pasted image 20261024202301.png]]

```
mr. meeseek hair
```

### 2º ingrediente

Siguiendo la pista de `clue.txt`, miramos en el sistema de archivos. En `/home` está el directorio del usuario `rick` y dentro, un archivo con espacio en el nombre (hay que escaparlo con `\` o usar comillas):

```
cd /home/
ls
cd rick
ls
cat 'second ingredients'
```

![[Pasted image 20261024202406.png]]

```
1 jerry tear
```

### 3er ingrediente

Lo que queda es la casa de root. `ls` normal no muestra nada, pero ya somos root:

```
cd /
cd root
ls
cat 3rd.txt
```

![[Pasted image 20261024202204.png]]

![[Pasted image 20261024202146.png]]

```
3rd ingredients: fleeb juice
```

## Respuestas de la sala

| Ingrediente | Respuesta |
|---|---|
| 1º | `mr. meeseek hair` |
| 2º | `1 jerry tear` |
| 3º | `fleeb juice` |

Con esto Rick vuelve a humano y terminamos la máquina.
