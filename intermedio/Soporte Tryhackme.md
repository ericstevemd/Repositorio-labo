# Soporte

Máquina de nivel intermedio de TryHackMe. IP: `10.65.184.240`

## 1. Reconocimiento

El primer paso es el reconocimiento con `nmap` para ver puertos abiertos y versiones de los servicios:

```bash
nmap -sV -sC 10.65.184.240
```

![[Pasted image 20260923142023.png]]

Se abren los siguientes puertos:
- **22/tcp** → OpenSSH 9.6p1 Ubuntu (protocol 2.0).
- **80/tcp** → Apache httpd 2.4.58 (Ubuntu), con el título *Support Operations Panel*.

Además, `nmap` detecta que la cookie `PHPSESSID` no tiene la bandera `HttpOnly` y que el sistema operativo es Linux.

## 2. Enumeración web

Con el puerto 80 abierto, enumeramos rutas y archivos con `gobuster`:

```bash
gobuster dir -u http://10.64.139.243 -w /usr/share/wordlists/dirb/common.txt
```

![[Pasted image 20260923143220.png]]

Entre los resultados más interesantes aparecen:
- `/index.php` → 200.
- `/api.php` → 302 (API interna de la aplicación).
- `/config.php` → 200.
- `/dashboard.php` → 302.
- `/logout.php`, `/info.php`, `/footer.php`, `/includes`, `/layout`, `/js` y `/skins`.

La enumeración no nos da nada explotable de forma directa, así que pasamos a atacar el formulario de login por fuerza bruta. Para ello contamos con el correo electrónico `help@support.thm`.

## 3. Fuerza bruta del login con hydra

Usamos `hydra` con el módulo `http-post-form` contra el formulario de login:

```bash
hydra -l help@support.thm -P /usr/share/wordlists/rockyou.txt 10.65.184.240 http-post-form "/:email=^USER^&password=^PASS^:F=Invalid credentials"
```

Análisis del comando:
- `-l help@support.thm` → correo fijo que probamos como usuario.
- `-P .../rockyou.txt` → diccionario de contraseñas a probar.
- `http-post-form` recibe tres campos separados por dos puntos:
  - **Ruta**: `/` (la raíz del sitio).
  - **Cuerpo de la petición**: `email=^USER^&password=^PASS^`, donde `^USER^` y `^PASS^` se sustituyen por cada combinación de usuario y contraseña.
  - **Cadena de fallo**: `F=Invalid credentials` es el texto que devuelve el servidor cuando el login es incorrecto; mientras hydra lo vea en la respuesta, sigue probando con la siguiente contraseña.

![[Pasted image 20260923143858.png]]

Hydra encontró una contraseña válida:
- **Usuario**: `help@support.thm`
- **Contraseña**: `snoopy`

## 4. Manipulación de la cookie `isITUser`

Con esas credenciales entramos en la web y accedemos al panel:

![[Pasted image 20260923144327.png]]

En la parte de la cookie encontramos un fallo de seguridad. Abrimos las herramientas de desarrollo (Storage → Cookies) y vemos el parámetro `isITUser`, cuyo valor está almacenado en MD5:

![[Pasted image 20260923144637.png]]

El valor completo del parámetro es `68934a3e9455fa72420237eb05902327`. Al pasarla por un servicio de cracking de hashes vemos que es el hash MD5 de `false`:

![[Pasted image 20260923144811.png]]

Como el parámetro está en MD5, cambiamos el valor de falso a `true` para hacernos pasar por usuario de TI. Para obtener el hash usamos CyberChef con la operación MD5 sobre la cadena `true`:

https://gchq.github.io/CyberChef/#recipe=MD5()&input=dHJ1ZQ

![[Pasted image 20260923145103.png]]

El resultado es `b326b5062b2f0e69046810717534cb09`.

Copiamos ese hash y lo pegamos en la cookie `isITUser`, sustituyendo al de `false`, y recargamos la página:

![[Pasted image 20260923145420.png]]

El panel ya muestra el bloque **IT Admin Panel** con el botón *View API*, algo que antes no aparecía.

## 5. IDOR en la API: `/user/3` → `/user/1`

Al entrar en la API vemos que nos indica la ruta de nuestro perfil:

![[Pasted image 20260923145502.png]]

- Aviso: *As a helpdesk user, you can query your own profile: /user/3*
- Petición: `GET /user/3`
- Respuesta JSON con `email: "help@support.thm"`, `2FA: false` y `admin: false`.

Como el identificador está expuesto en la URL, probamos un IDOR cambiando `/user/3` por `/user/1`:

![[Pasted image 20260923145550.png]]

La respuesta del usuario 1 es:

```json
{
  "email": "specialadmin@support.thm",
  "2FA": false,
  "admin": true
}
```

De este modo obtenemos el correo del administrador: `specialadmin@support.thm`.

## 6. Lectura de `/config`

Ahora volvemos al tablero y entramos en la parte de configuración para ver `./config`. Para ello manipulamos el parámetro `skin` de la URL (que tomaba valores como `green`) y forzamos la ruta al archivo de configuración:

```
dashboard.php?skin=../config
```

![[Pasted image 20260923145709.png]]

En el código fuente de la configuración encontramos la clave del administrador:

![[Pasted image 20260923132039.png]]

```php
$MASTER_PASSWORD = 'support110';
$SITE_VER = '1.0';
$SITE_NAME = 'support_portal';
```

## 7. Login de administrador

Con la contraseña maestra intentamos ingresar con las credenciales del administrador:
- **Usuario**: `specialadmin@support.thm`
- **Contraseña**: `support110` (en la nota aparece anotada también como `support@110`).

Al autenticarnos correctamente vemos la bandera de administrador:

![[Pasted image 20260923150012.png]]

El panel muestra *Administrator Access Confirmed* → **THM{I_AM_ADMIN999}**.

## 8. RCE en el campo de fecha

Una vez ingresado como administrador vemos en el dashboard el selector de tema y el campo de fecha (*Date*). Ese campo envía al servidor el parámetro `sys`, que se ejecuta en la máquina, por lo que podemos inyectar un comando concatenándolo con `;`:

```
sys=date;cat /home/ubuntu/user.txt
```

![[Pasted image 20260923150340.png]]

La respuesta del servidor devuelve primero la salida de `date` (`Wed Sep 23 20:03:01 UTC 2026`) y, debajo, el contenido de `/home/ubuntu/user.txt`.

## 9. Bandera

- **Administrador**: `THM{I_AM_ADMIN999}`
- **Usuario (user.txt)**: `THM{GOT_THE_FLAG001}`
