
como primer paso podemos realizar  reconocimiento 

nmap -p- --open  --min-rate 5000 -sS  -Pn  -n -vvv 192.168.100.161

![[Captura de pantalla 2026-09-29 234606 1.png]]

![[Pasted image 20260929234333.png]]
utilizamos el siguiente comando 
smbclient -L  //192.168.100.168 -N 

![[Captura de pantalla 2026-09-29 235328.png]]

por los cual ingresamos  a menu y encotramos un archivo ocultto 

![[Pasted image 20260930011540.png]]
y por lo cual podemos  encontrar 
user = marmai
pass = EspabilaSantiago69

una vez auntenida la credenciales 
y intenteamos ingresar por el puerto 10021  con el ftp 


una vez adentro  descargamos un archivo zip 
![[Pasted image 20260930014832.png]]

una vez descargado podemos realizar los siguiente  sacamos el hash de archivo zip 
y lo bolcarmos en un archivo 
![[Pasted image 20260930014857.png]]
y podemos observar  el hash del  nuestros archivo zip  

![[Pasted image 20260930014914.png]]
y intentamos adivinar la clave  con john rippet  con una lista de diccionario y encontramos la credenciales  de nuestros archivo zip 

![[Pasted image 20260930014938.png]]

introducíos la credenciales para pode entra  al archivo 
![[Pasted image 20260930014956.png]]
y revisamos la lista de usuario 

![[Pasted image 20260930015031.png]]

y revisamos los archivo para verificar el contenido 
![[Pasted image 20260930015054.png]]
y hacemos los mismos volacamos para poder ingresar hash de id_rsa obtener la credenciales 

![[Pasted image 20260930015136.png]]
y utilizamos la herramienta hydra para encontrar la credenciales  de la lista de usuario 

 ![[Pasted image 20260930015202.png]]

login: gurpreet  
password: babygirl
y como podemos ver tenemos los siguiente archivo 

![[Pasted image 20260930015320.png]]
y revisamos la cat user.txt  
![[Pasted image 20260930015342.png]]

realizamos conexiones a la base de datos  con mariadb 


mariadb -u gurpreet -p

una ves ingresador 
show databases ;

![[Pasted image 20260930153031.png]]

![[Pasted image 20260930155619.png]]

![[Pasted image 20260930155825.png]]

![[Pasted image 20260930155854.png]]
y vemos que tenemos usuario y contraseña 
  1 | carline | 703ff9a12582b2aaaa3fe7f89bb976c8 |
|  2 | nika    | c6f606a66a30cbaa428131d4c074787
por local podemos revisar  encontramos credenciales  para ingresar  

la contraseña  de nika:lucymylove

![[Pasted image 20260930021032.png]]

ingresamos por ssh  a los usuario que corresponde 


![[Pasted image 20260930163736.png]]


echo "/bin/bash" > find

chmod 777 find
echo $PATH
/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games

 P
![[Pasted image 20260930170612.png]]

sudo PATH=/opt/porno:$PATH /opt/porno/watchporn.sh 

![[Pasted image 20260930170730.png]]

![[Pasted image 20260930170954.png]]