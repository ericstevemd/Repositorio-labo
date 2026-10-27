# Enumeración web

Comandos para descubrir dominios, subdominios, directorios y archivos en los servicios web.

## 1. Configurar el dominio en /etc/hosts

Para acceder a un dominio virtual que solo existe en la máquina objetivo:

```
echo '<IP> <nombre-del-dominio>' >> /etc/hosts
```

O editando el archivo directamente:

```
nano /etc/hosts
```

## 2. Identificar la tecnología (whatweb)

```
whatweb http://<dominio>:<puerto>
```

Devuelve el servidor, el CMS, las versiones y el título (por ejemplo, Jenkins sobre Jetty).

## 3. Directorios y archivos (gobuster)

```
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt -t 50
```

Con extensión de archivos concretos:

```
gobuster dir -u http://<IP>/wordpress -w /usr/share/wordlists/dirb/common.txt -t 50 -x php,txt,html
```

- `-u` → URL objetivo.
- `-w` → diccionario.
- `-t` → hilos.
- `-x` → extensiones a probar.

Con una wordlist más grande y muchas extensiones (búsqueda exhaustiva):

```
gobuster dir -u http://<IP> -w /home/kali/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x asp,aspx,txt,html,php,bak,old,config,json,xml
```

- El diccionario `DirBuster-2007_directory-list-2.3-medium.txt` es mucho más grande que `common.txt`, tarda bastante más pero cubre muchas más rutas.
- Las extensiones `bak`, `old` y `config` suelen revelar copias de seguridad o ficheros de configuración filtrados.

También se pueden enumerar APIs con diccionarios específicos de SecLists:

```
gobuster dir -u http://<IP>:12345 -w /home/kali/SecLists/Discovery/Web-Content/api/api-endpoints-res.txt
```

## 4. Subdominios (wfuzz con cabecera Host)

```
wfuzz -c \
  -w /home/kali/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
  -H "Host: FUZZ.<dominio>" \
  --hl 363 \
  http://<dominio>/
```

- `-c` → colorear la salida.
- `FUZZ` → marcador que se sustituye por cada palabra del diccionario.
- `--hl` / `--hh` / `--hw` → descartan respuestas con ese número de líneas, bytes o palabras (evita falsos positivos).

## 5. Ficheros ocultos (ffuf)

```
ffuf -u http://<IP>/FUZZ -w /home/kali/SecLists/Discovery/Web-Content/raft-large-files-lowercase.txt -c -r -t 150 -ic -fw 20
```

- `-ic` → ignora certificados SSL inválidos.
- `-fw 20` → filtra respuestas de 20 palabras (páginas de error genéricas).
- `-r` → sigue las redirecciones.

## 6. Peticiones manualmente (curl)

```
curl -s "http://<IP>/ruta?q=1"
```

Con cabecera personalizada:

```
curl -s -H "Authorization: Bearer <TOKEN>" "http://<IP>/api/courses.php?q=1"
```

## 7. Enumerar una API en bucle (curl + grep)

```bash
#!/bin/bash
for i in {1..25}; do
  curl -s -X GET "http://<IP>:12345/user/$i" | grep username | cut -d\" -f4 >> usernames.txt
done
```

- `grep username` → filtra el campo.
- `cut -d\" -f4` → extrae el valor entre comillas.

## 8. Diccionario propio con CeWL

Genera una lista de palabras a partir del contenido de una página web:

```
cewl -d 2 -m 7 --lowercase --with-numbers -w cewl_words.txt http://<dominio>:<puerto>
```

- `-d 2` → profundidad de rastreo.
- `-m 7` → longitud mínima.
- `--lowercase` → todo en minúsculas.
- `--with-numbers` → incluye palabras con números.

## 9. Diccionarios usados en este repo

- `/usr/share/wordlists/dirb/common.txt`
- `/home/kali/SecLists/Discovery/Web-Content/common.txt`
- `/home/kali/SecLists/Discovery/Web-Content/raft-large-files-lowercase.txt`
- `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`
- `/home/kali/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt`
- `/home/kali/SecLists/Discovery/DNS/subdomains-top1million-5000.txt`
- `/usr/share/wordlists/rockyou.txt` (contraseñas)
