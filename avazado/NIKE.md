# NIKE

Máquina de nivel avanzado. IP: `192.168.33.102`

## 1. Reconocimiento

Escaneamos la máquina con `nmap -sV -sC`:

```
nmap -sV -sC 192.168.33.102
```

![[Pasted image 20261005154727.png]]

Se abren los puertos:
- **22/tcp** → OpenSSH 9.2p1 (Debian).
- **80/tcp** → Apache httpd 2.4.62, con el título **Tienda Nike - Zapatillas**.

## 2. Enumeración de archivos

Fuzzing de archivos ocultos en el servidor web con `ffuf` y el diccionario `raft-large-files-lowercase.txt` (filtrando las respuestas con 20 palabras, típicas de las páginas de error):

```
ffuf -u http://192.168.33.102/FUZZ -w /home/kali/SecLists/Discovery/Web-Content/raft-large-files-lowercase.txt -c -r -t 150 -ic -fw 20
```

![[Pasted image 20261005164105.png]]

Entre los resultados aparecen `index.html` y `styles.css` (archivos normales de la tienda) y **`upload.php`**, un posible formulario de subida de archivos que conviene investigar.
