# PapaFirta

Máquina de nivel principiante de The HackersLab. IP: `192.168.100.160`

## 1. Reconocimiento

Escaneamos los puertos y servicios de la máquina con `nmap`:

```
nmap -sV -sC 192.168.100.160
```

![[Pasted image 20260929012550.png]]

Se abren dos puertos:
- **22/tcp** → OpenSSH 9.2p1 (Debian 2+deb12u2).
- **80/tcp** → Apache httpd 2.4.57 (Debian).

La máquina es una VirtualBox con Linux, así que el vector de entrada más probable es la web.

## 2. Revisión de la página web

Visitamos `http://192.168.100.160/` y vemos la página por defecto de Apache, sin nada interesante en pantalla:

![[Captura de pantalla 2026-09-29 012802.png]]

Como la cara de la web no aporta nada, revisamos el código fuente con `view-source:` y ahí aparece algo raro: un bloque de comentarios HTML que oculta una larga secuencia de caracteres típicos de Brainfuck:

![[Captura de pantalla 2026-09-29 012625.png]]

## 3. Código en Brainfuck en los comentarios

Esas líneas escondidas dentro del comentario no son HTML basura: están escritas en **Brainfuck**, así que vamos a decodificarlas para ver qué esconden.

## 4. Decodificación con dcode.fr

Usamos la web [https://www.dcode.fr/](https://www.dcode.fr/) y su intérprete de Brainfuck para desentrañar el código:

![[Pasted image 20260929094836.png]]

Al ejecutarlo, la salida es:

```
abuelacalientalasopa
```

## 5. Diccionario de usuarios

La cadena decodificada no es aleatoria: la partimos en tres palabras y creamos nuestro diccionario de usuarios para intentar entrar en la máquina:

- `abuela`
- `calienta`
- `lasopa`

Guardamos esos usuarios en el fichero `cliente.txt` y usaremos `abuelacalientalasopa` como contraseña candidata.

## 6. Fuerza bruta SSH con hydra

Atacamos el servicio SSH con `hydra`, pasando la lista de usuarios y la contraseña única:

```
hydra -L /home/kali/cliente.txt -p abuelacalientalasopa ssh://192.168.100.160
```

![[Pasted image 20260929012513.png]]

Hydra valida el par en un solo intento:

- **login:** `abuela`
- **password:** `abuelacalientalasopa`

Nos autenticamos por SSH con esas credenciales.

## 7. Escalada de privilegios con node

Una vez dentro como `abuela`, enumeramos el directorio home: existe `user.txt`, pero `cat user.txt` devuelve *Permiso denegado*. Comprobamos los permisos con `sudo -l`:

![[Pasted image 20260929012315.png]]

`abuela` puede ejecutar `/usr/bin/node` como root sin contraseña (`(root) NOPASSWD: /usr/bin/node`).

Buscamos en [GTFOBins](https://gtfobins.github.io/gtfobins/node/#sudo) cómo aprovechar ese binario para escalar. Una ejecución de `node -e '...'` sin `sudo` se queda en el usuario `abuela`; la forma correcta es lanzar la shell interactiva de node con privilegios de root:

```
sudo node -e 'require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'
```

![[Pasted image 20260929012426.png]]

Con `id` confirmamos que somos root: `uid=0(root) gid=0(root) grupos=0(root)`.

## 8. Banderas

Ya con la sesión de root, leemos las banderas de la máquina:

```
cat user.txt
cat root.txt
```

Con esto completamos la máquina **PapaFirta**.
