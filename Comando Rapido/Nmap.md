# Nmap — escaneo de puertos

Comandos usados en las notas de este repositorio para el reconocimiento inicial.

## 1. Escaneo básico (versión de servicios y scripts por defecto)

```
nmap -sV -sC <IP>
```

- `-sV` → detecta la versión de cada servicio.
- `-sC` → ejecuta los scripts por defecto de Nmap (NSE).

## 2. Todos los puertos, rápido

```
nmap -p- --open --min-rate 5000 -sS -Pn -n -vvv <IP>
```

- `-p-` → escanea los 65535 puertos.
- `--open` → solo muestra los que están abiertos.
- `--min-rate 5000` → fuerza un mínimo de paquetes por segundo.
- `-sS` → escaneo SYN (stealth).
- `-Pn` → no hace ping, trata el host como si estuviera vivo.
- `-n` → no resuelve DNS (más rápido).
- `-vvv` → máxima verbosidad.

## 3. Escaneo completo con guardado de resultados

```
nmap -p- --open -sV -sC -sS -T4 -n -Pn -vvv <IP> -oG open_port.txt
```

- `-T4` → timing agresivo (T0–T5).
- `-oG open_port.txt` → guarda el resultado en formato *grepable* para filtrarlo después:

```
grep open open_port.txt
```

## 4. Puertos y servicios que suelen aparecer en estas máquinas

| Puerto | Servicio | Qué mirar |
| --- | --- | --- |
| 21/tcp | FTP | Acceso anónimo (`anonymous`) |
| 22/tcp | SSH | Fuerza bruta con `hydra` |
| 80 / 443 / 8080 / 6969 | HTTP | Directorios ocultos, Jenkins, paneles |
| 139 / 445/tcp | SMB | Recursos compartidos sin autenticación |
| 389 / 636 / 3268/tcp | LDAP | Dominio de Active Directory |
| 88/tcp | Kerberos | Enumeración de usuarios (`kerbrute`) |
| 3306/tcp | MySQL | Credenciales por defecto o mal guardadas |
| 5985 / 47001/tcp | WinRM | Acceso con `evil-winrm` |

## 5. Scripts útiles por servicio

```
nmap -p 445 --script=smb-enum-shares,smb-os-discovery <IP>
nmap -p 389 --script=ldap-rootdse <IP>
nmap -p 80,443 --script=http-title,http-headers <IP>
```

Los scripts por defecto (`-sC`) ya devuelven información útil como `nbstat` (nombre NetBIOS), `smb2-time` y el *clock skew* del dominio.
