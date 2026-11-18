# TryHackMe — Interceptor (MediaHub)

**Sala:** Interceptor
**Plataforma:** TryHackMe
**Dificultad:** Media
**Tema:** Enumeración web · Información filtrada en el código fuente · Login con 2FA · **Manipulación de peticiones con Burp/DevTools** (bypass de OTP) · Reverse shell
**Objetivo:** Iniciar sesión como administrador en el portal *MediaHub* y leer `/var/www/user.txt`.

> **Resumen en una línea:** `nmap` y `gobuster` dejan al descubierto el portal **MediaHub** (`/phpmyadmin`, `/uploads`); una **nota de desarrollador olvidada en el código PHP** revela el correo del admin y el formato de su contraseña; el login tiene **OTP de 6 dígitos**, pero `verify_otp.php` confía en un flag del propio cliente (`is_verified`) que se manipula con **Burp** → acceso admin → flag 1; desde ahí, reverse shell → `/var/www/user.txt` → flag 2.

---

## 0. Información de la máquina

- **IP:** varía con el despliegue (en las capturas: `10.64.156.32` en el reconocimiento y `10.66.190.111` en la explotación). Usa tu IP actual.
- **Servicios:**

| Puerto | Servicio | Detalle |
| --- | --- | --- |
| 22/tcp | SSH | OpenSSH 8.2p1 (Ubuntu 4ubuntu0.7) |
| 53/tcp | DNS | ISC BIND 9.16.1-ubuntu |
| 80/tcp | HTTP | Apache 2.4.41 — **MediaHub** (`http-title: MediaHub`), PHP con `PHPSESSID` **sin flag `httponly`** |

---

## 1. Reconocimiento (Nmap)

```bash
nmap -sC -sV IP_MACHINA
```

![[Pasted image 20261117222015.png]]

- El `http-title: MediaHub` da el nombre de la aplicación y del dominio interno (`mediahub.thm`).
- `PHPSESSID: flag not set` → la cookie no tiene `HttpOnly` (preparada para robo de sesión, aunque aquí no hizo falta).

---

## 2. Enumeración de directorios (gobuster)

```bash
gobuster dir -u http://IP_MACHINA -w /home/kali/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -b 200
```

![[Pasted image 20261118001429.png]]

Directorios interesantes:

- `/uploads` → **301** (posible subida de ficheros).
- `/phpmyadmin` → **301** (panel de base de datos).
- `/assets`, `/javascript` → estáticos.
- `/server-status` → 403.

---

## 3. Nota de desarrollador olvidada (revisión del código)

En el código fuente de una de las páginas (view-source / fichero `.php`) hay un comentario que no se borró antes de desplegar:

```php
/*
|----------------------------------------------------
| Developer Note (temporary)
|----------------------------------------------------
| Admin test account for staging environment
| Email: admin@mediahub.thm
|
| Password policy reminder:
| Admin password follows company format:
| MediaHub + any year
| TODO: remove before production deployment
*/
```

![[Pasted image 20261118001354.png]]

**Pista obtenida:**

- **Usuario admin:** `admin@mediahub.thm`
- **Contraseña:** `MediaHub` + **cualquier año** (`MediaHub2024`, `MediaHub2025`, `MediaHub2026`…) — probar los últimos años a mano en el login.

> Lección: **los comentarios en el código fuente son información de primera clase** — no hace falta SQLi si el desarrollador dejó las credenciales escritas.

---

## 4. Login y el muro de 2FA

![[Pasted image 20261118022410.png]]

Con `admin@mediahub.thm` + la contraseña correcta, el servidor no deja pasar: manda a **`otp.php`**, que pide un **OTP de 6 dígitos** (*Two-Factor Verification*):

![[Pasted image 20261118022524.png]]

Endpoints vistos en la pestaña *Network*:

- `POST /api_login.php` → `multipart/form-data` con `email` y `password` (aquí se validan).
- `POST /verify_otp.php` → recibe el campo `otp` y responde JSON.

---

## 5. Bypass del OTP (manipulación de la petición)

Probando con un OTP falso (`123456`), la respuesta de `verify_otp.php` es:

```json
{ "ok": false, "error": "Invalid OTP. Try again.", "is_verified": false }
```

![[Pasted image 20261118022613.png]]

El nombre del campo **`is_verified`** delata que la validación se confía al **cliente**. Hay dos maneras de saltarlo (la sala apunta a **Burp**):

1. **Manipular la petición:** añadir al *body* multipart un campo extra `is_verified=true` junto al `otp`:

```
------gecko...
Content-Disposition: form-data; name="otp"

123456

------gecko...
Content-Disposition: form-data; name="is_verified"

true
------gecko...--
```

![[Pasted image 20261118022712.png]]

2. **Manipular la respuesta** (en Burp, *Intercept → respuesta del servidor*): cambiar `ok: false` → `true` e `is_verified: false` → `true`; el frontend acepta y te deja entrar.

Cualquiera de las dos lleva al dashboard de administración, donde aparece el **indicador con la flag**:

**Flag 1 → `THM{ADMIN_ACCESS_USING_BURP}`**

> En las capturas se ve el intento con DevTools; en la sala se resuelve con **Burp Suite** (Proxy → *Intercept on* → editar petición/respuesta → *Forward*).

---

## 6. De admin a shell — `/var/www/user.txt`

Con la sesión de admin, el vector es la **subida de ficheros** (el directorio `/uploads` que ya apareció en el gobuster): se sube una webshell/reverse shell y se lanza desde el navegador (tab *Online – Reverse Shell Generator* abierto en las capturas = [revshells.com](https://revshells.com)).

Con shell como `www-data`:

```bash
cat /var/www/user.txt
```

**Flag 2 → `THM{SYSTEM_PWNED_SUCCESSFULLY}`**

> ⚠️ **Pendiente de completar:** las capturas de esta fase (subida del shell, reverse shell y lectura de `user.txt`) se pegaron en un momento en el que los ficheros quedaron **vacíos (0 bytes)** y se perdieron — 6 imágenes. Si las repites, las integro aquí con el detalle exacto (ruta de subida, payload usado y cómo se lanzó).

**Preguntas de la sala (respondidas correctamente):**

| Pregunta | Respuesta |
| --- | --- |
| Valor del indicador después de iniciar sesión como administrador | `THM{ADMIN_ACCESS_USING_BURP}` |
| Valor de `/var/www/user.txt` | `THM{SYSTEM_PWNED_SUCCESSFULLY}` |

![[Pasted image 20261118021804.png]]

---

## 7. Flags y lecciones

| Flag | Valor | Dónde |
| --- | --- | --- |
| Admin | `THM{ADMIN_ACCESS_USING_BURP}` | indicador tras el login admin |
| Usuario | `THM{SYSTEM_PWNED_SUCCESSFULLY}` | `/var/www/user.txt` |

1. **Los comentarios del código fuente filtran credenciales**: una "nota temporal" de desarrollador dio el correo y el formato de la contraseña.
2. **La 2FA no sirve de nada si el cliente manda el resultado**: `is_verified` (o `ok`) en la respuesta/petición siempre se debe validar **en el servidor**.
3. **Burp es la herramienta de la sala**: interceptar y editar tanto peticiones como respuestas (DevTools sirve para mirar, Burp para manipular).
4. **`/uploads` + PHP = combinación clásica** para pasar de panel a shell.
5. El `nmap` mostró el `httponly` ausente en `PHPSESSID` — pista de que las cookies aquí eran *in-game* manipulables.

> Comandos relacionados: `Comando Rapido/Enumeracion web.md`, `Comando Rapido/Shells y transferencia.md`, `Comando Rapido/Fuerza bruta y crackeo.md`.
