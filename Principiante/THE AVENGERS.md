# THE AVENGERS

Máquina de nivel principiante de The HackersLab. IP: `192.168.100.153`

## 1. Reconocimiento

Escaneamos los puertos y servicios de la máquina con `nmap`:

```
nmap -sV -sC 192.168.100.153
```

![[Captura de pantalla 2026-09-27 235314.png]]

Se abren los siguientes puertos:

- **21/tcp** → vsftpd 3.0.5, con acceso anónimo permitido (`ftp-anon: Anonymous FTP login allowed`).
- **22/tcp** → OpenSSH 8.9p1 (Ubuntu).
- **80/tcp** → Apache httpd 2.4.52, con el título *Avengers Hacking Ético* y un `robots.txt` que desvela las rutas `/webs/` y `/mysql/`.
- **3306/tcp** → MySQL 8.0.36.

## 2. Enumeración FTP anónima

El escaneo confirma que el FTP acepta login anónimo, así que nos conectamos con el usuario `anonymous`:

```
ftp anonymous@192.168.100.153
```

Una vez dentro, el listado en modo pasivo muestra dos archivos que descargamos con `get`:

```
get FLAG.txt
get credential_mysql.txt.zip
```

![[Pasted image 20260928000450.png]]

El archivo `credential_mysql.txt.zip` está protegido con contraseña, así que de momento solo nos quedamos con los archivos descargados.

## 3. Enumeración web con gobuster

Enumeramos la página web con `gobuster` y el diccionario `common.txt` de SecLists:

```
gobuster dir -u http://192.168.100.153 -w /home/kali/SecLists/Discovery/Web-Content/common.txt
```

![[Pasted image 20260928001850.png]]

Además de `index.html` y `robots.txt`, aparecen varios directorios interesantes: `/code`, `/css`, `/flags`, `/mysql` y `/php`.

## 4. Hallazgos: contraseñas ocultas y el usuario Hulk

Dentro del directorio `/mysql/` encontramos la página *Base de Datos MySQL*:

![[Pasted image 20260928002145.png]]

Al ver el código fuente de `http://192.168.100.153/mysql/database.html` aparecen comentarios HTML ocultos con una contraseña codificada en Base64:

![[Pasted image 20260928002214.png]]

Decodificamos la cadena con un decodificador de Base64 y obtenemos `fuerzabruta`:

![[Pasted image 20260928004326.png]]

Usamos ese valor en el buscador de la web secreta (*Web Secreta 1*):

![[Pasted image 20260928004301.png]]

El buscador responde «¡Has encontrado a Hulk!», con lo que identificamos el nombre de usuario: `hulk`.

## 5. Acceso SSH como hulk

Nos autenticamos por SSH con el usuario `hulk`:

```
ssh hulk@192.168.100.153
```

![[Pasted image 20260928011234.png]]

El servidor acepta la conexión y recibimos el banner de bienvenida de Ubuntu 22.04.4 LTS.

## 6. Enumeración como hulk

Listamos el directorio personal con `tree`:

```
tree
```

![[Pasted image 20260928011413.png]]

Encontramos una cantidad llamativa de directorios y archivos con nombres engañosos (`NO_FLAG.txt`, `burro`, `decrypt.txt`, `user.txt`...). Dentro de `mysql/hint/zip/` aparece el archivo `shit_how_they_did_know_this_password.txt`, que leemos con `cat`:

```
cat shit_how_they_did_know_this_password.txt
```

![[Pasted image 20260928011435.png]]

El archivo muestra un mensaje de felicitación: hemos dado con la contraseña que descifra el `credential_mysql.txt.zip` descargado por FTP (el propio archivo insinúa que hay que mirarlo con atención para verla).

## 7. Fuerza bruta del sufijo numérico

Una vez descifrado el zip, examinamos su contenido con `cat`:

```
cat credential_mysql.txt
```

![[Pasted image 20260928011507.png]]

El archivo indica que la contraseña del usuario MySQL es `fuerzabrutaXXXX`, donde `XXXX` es un número que está en el rango 0-3000. Al desconocer los dígitos, recurrimos a la fuerza bruta sobre ese rango hasta dar con la contraseña final.

## 8. Credenciales finales

- **Usuario:** `hulk`
- **Contraseña:** `fuerzabrutaxxx`

> Pendiente de documentar la post-explotación y la escalada de privilegios.
