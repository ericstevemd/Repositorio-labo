# Comandos básicos

Comandos fundamentales para empezar cualquier laboratorio.

## 1. Conexión SSH

```
ssh <usuario>@<IP>
```

Ejemplo real:

```
ssh asrepuser1@10.211.12.20
```

- Pide la contraseña del usuario al conectar.
- Primera conexión: pregunta `Are you sure you want to continue?` → escribir `yes`.
- Puerto distinto al 22: `ssh <usuario>@<IP> -p <PUERTO>`.

**Copiar archivos con SCP** (misma conexión que ssh):

```
scp <usuario>@<IP>:/ruta/archivo.txt .      # baja el archivo
scp archivo.exe <usuario>@<IP>:/ruta/       # sube el archivo
```

## 2. Windows — ayuda de los comandos net

```
net help
```

Lista todos los comandos `net` disponibles (`net user`, `net group`, `net share`, `net view`, `net localgroup`...).

Para ver la sintaxis de uno concreto:

```
net help user
net help group
```

**Cambiar la contraseña de un usuario de dominio** (`net rpc`, desde Linux) — usado en **Restablecer** para recorrer la cadena `SHAWNA_BRAY → CRUZ_HALL → DARLA_WINTERS`:

```
net rpc password "<usuario>" "<nueva_password>" -U "<DOMINIO>"/"<usuario_actual>"%'<password_actual>' -S "<DOMINIO>"
```

Ejemplo real:

```
net rpc password "SHAWNA_BRAY" "newP@ssw0rd2022" -U "THM.CORP"/"TABATHA_BRITT"%'marlboro(1985)' -S "THM.CORP"
```

- Cada eslabón de la cadena usa la credencial que acabas de cambiar en el paso anterior.
- Es el equivalente por línea de comandos a lo que hace `ForceChangePassword` en BloodHound.

## 3. FTP — acceso anónimo

```
ftp anonymous@<IP>
```

Dentro del `ftp>`:

```
ls
cd pub
ls
get archivo.tar.gz
bye
```

- Si el `nmap` muestra `Anonymous FTP login allowed`, probarlo siempre: en **Operation Coldstart** `pub/backup.tar.gz` contenía el código completo de la aplicación.
- Sin shell interactiva: `wget ftp://<IP>/pub/archivo.tar.gz`.

## 4. Telnet — leer banners de servicios

Conectarse a un puerto y leer lo que el servicio saluda (**banner grabbing**), sin autenticar. En **Net Sec Challenge** las flags estaban justo ahí:

```
telnet <IP> 22
```

Salida típica (SSH se anuncia al conectar):

```
SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.7
```

Para un servicio HTTP hay que pedirle algo a mano (escribir las líneas y cerrar con una línea en blanco):

```
telnet <IP> 80
GET / HTTP/1.1
Host: tryhackme.com
```

- La cabecera `Server:` de la respuesta es la que suele llevar datos interesantes (versión e incluso flags en CTF).
- Alternativa más cómoda: `curl -I http://<IP>/` (mismas cabeceras, solo cabecera HTTP).
- Salir de telnet: `Ctrl+]` y después `quit`.

## 5. Consultas DNS básicas

Cuando el `nmap` muestra el puerto 53 (`ISC BIND`, como en **Interceptor**) o la sala menciona un dominio:

```
dig <dominio>                 # respuesta completa
dig <dominio> +short          # solo la IP
dig <dominio> @<IP_DNS>       # contra el servidor DNS de la máquina
nslookup <dominio>            # versión simple (también en Windows)
host <dominio>                # una línea por registro
```

- Registro interesante: `dig <dominio> ANY +short` (todos los registros que el servidor quiera listar).
- Zonas invertidas: `dig -x <IP>` (qué nombre corresponde a esa IP).

## 6. Extraer ficheros comprimidos

```
tar -xvf archivo.tar        # .tar
tar -xzvf archivo.tar.gz    # .tar.gz (gzip)
unzip archivo.zip           # .zip
zip -r salida.zip carpeta/  # comprimir
```

- Letras: `x` extraer · `z` gzip · `v` verbose (va bien para ver qué saca) · `f` fichero.
- Ejemplos del repo: `unzip BurgerWithoutCheese.zip` (**GOIKO**), `unzip documents.zip` (**boda**), `tar -xvf backup.tar.gz` (**Operation Coldstart**, el backup que contenía `app.py`).

## 7. Leer un fichero sin `cat`

En paneles web que bloquean `cat` (*Command disabled...*) o cuando el fichero es un binario:

```
less archivo        # con scroll y buscador (/ para buscar)
tac archivo         # el contenido del revés
head -n 20 archivo  # primeras líneas
tail -n 20 archivo  # últimas líneas
grep "" archivo     # vuelca todo el contenido
strings binario     # extrae las cadenas legibles
```

- Ejemplo real: **Try Hack Me - Pickle Rick** bloqueaba `cat` y la flag salió con `less`/`head`/`strings`.
- `strings` sirve también para cazar flags/URLs dentro de binarios o firmware.

## 8. Codificar y descodificar (base64)

Cookies, cabeceras y JWT vienen en base64 (**Domino**, **Vaucher**):

```
echo -n 'texto' | base64 -w0        # codificar (sin salto de línea final)
echo 'dGV4dA==' | base64 -d         # descodificar
```

- `-n` evita el salto final y `-w0` evita que lo parta en líneas de 76 caracteres (importante al re-codificar tokens).
- El segmento del medio de un **JWT** es base64*url* (sin `=`): si `base64 -d` falla, rellena con `=` hasta completar múltiplo de 4.

## 9. Historial de comandos (nuestro y de la víctima)

```
history                 # historial de la sesión actual
cat ~/.bash_history     # historial guardado en disco
```

- En la víctima, el `.bash_history` de otros usuarios es un filón si es legible: en **BOLA**, el de `steven` revelaba credenciales.

## 10. Estado del sistema

```
df -h       # espacio en disco (¿hay sitio para subir binarios?)
free -m     # memoria
uptime      # carga y tiempo encendido
```

## 11. Leer ficheros y moverse por directorios en Windows

Desde `cmd` o PowerShell (no hay `cat`/`ls` de Linux) — usado en **Restablecer** para leer `user.txt` y `root.txt`:

```
type archivo.txt                              # equivalente a cat
type "C:\Users\administrator\Desktop\root.txt" # con ruta y espacios: entre comillas
dir                                           # equivalente a ls
cd C:\Users
cd Desktop
```

Ejemplo real:

```powershell
C:\Users\Administrator\Desktop>type root.txt
THM{RE_SET_AND_DELEGATE}
```

- Con espacios en la ruta hay que envolverla entre comillas dobles, si no `type` interpreta cada palabra como un argumento distinto.
- El equivalente en la shell de **evil-winrm** es el mismo `type`, porque ejecuta comandos de Windows.

## 12. Variables de entorno en Linux

Necesario en **Restablecer** para reutilizar el ticket Kerberos de Administrator:

```
export KRB5CCNAME=Administrator@cifs_HAYSTACK.THM.CORP@THM.CORP.ccache
echo $KRB5CCNAME        # leer su valor
unset KRB5CCNAME        # borrarla
```

- `export` define la variable **solo en la sesión actual** y la exporta a los procesos hijos (por eso el `impacket-wmiexec -k -no-pass` siguiente la encuentra).
- Sin `export` (poniendo solo `KRB5CCNAME=...`) la variable queda local al intérprete y las herramientas de Impacket no la ven.
- Se puede poner delante de un comando sin exportarla: `KRB5CCNAME=archivo.ccache impacket-wmiexec ...`.

## 13. Editar ficheros de texto con nano

```
nano /etc/hosts
```

- `Ctrl+O` guarda, `Ctrl+X` sale, `Ctrl+W` busca.
- En **Restablecer** se añadió `10.65.138.18 haystack.thm.corp` para que Kerberos resolviera el nombre del DC. La forma rápida de añadir la línea sin abrir el editor es `echo '<IP> <dominio>' >> /etc/hosts` (ver `Enumeracion web.md`).

> Nota: los comandos de *privesc* (`sudo find`, `ssh2john`…) viven en sus ficheros: `Escalada de privilegios Linux.md` y `Fuerza bruta y crackeo.md`. Los de Active Directory de **Restablecer** (`bloodhound-python`, `GetNPUsers`, `getST`, `wmiexec`, `impacket-lookupsid`, Responder…) están en `SMB y Active Directory.md` y `Fuerza bruta y crackeo.md`.
