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
