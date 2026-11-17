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
