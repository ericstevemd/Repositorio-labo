
primer paso podemos realizar el reconocimiento con la herramienta  nmap 
nmap -sV -sC 10.65.184.240
![[Pasted image 20260923142023.png]]

como segundo paso podemos realizar prueba  enumeración pagina web 
gobuster dir -u http://10.64.139.243 -w /usr/share/wordlists/dirb/common.txt

![[Pasted image 20260923143220.png]]
por los cual no entortamos nada por los cual realizamos por fuerza bruta  por cual tenemos el correo  electrónicos  help@support.thm

utilizamos la herramienta hydra para realizar explotacion por fuerza burta  

 hydra -l help@support.thm -P /usr/share/wordlists/rockyou.txt 10.65.184.240 http-post-form "/:email=^USER^&password=^PASS^:F=Invalid credentials"


![[Pasted image 20260923143858.png]]

por los cual encontramos  los siguiente usuario y clave  
 usuarios: help@support.thm
Clave: snoopy 

ingresamos en la pagina web 
![[Pasted image 20260923144327.png]]

y la parte de cooking  encontramos  un fallo de seguridad 
![[Pasted image 20260923144637.png]]

copiamos isITUser  y vemos que los siguiente no dar falso  para hacer administrador  
![[Pasted image 20260923144811.png]]
ahora realizamos vamos a realizar el cambio de falso a true  para hacer administrador   (https://gchq.github.io/CyberChef/#recipe=MD5()&input=dHJ1ZQ)
![[Pasted image 20260923145103.png]]

listo vamos realizar el cambio  a true copiamos y pegamos  en la cookis 
![[Pasted image 20260923145420.png]]

y vemos que una ruta  /user/3 

![[Pasted image 20260923145502.png]]

y nos cambiamos  /user/1  y encontramos el correo  de administrador 
![[Pasted image 20260923145550.png]]

y ahora damos a parte de tablero y entramos a configuración  para ver  ./config  
![[Pasted image 20260923145709.png]]

y encontramos  clave  de adminitrador 
![[Pasted image 20260923132039.png]]

por los cual intentamos ingresar  
usuario: specialadmin@support.thm
password: support@110 ---> para ingresar password  
support110
y ingresamos el usuarios 
encontramos flag  
![[Pasted image 20260923150012.png]]

una ver ingresado podemos ver  data o time  
para poder ejecutar  el siguiente comando   sys=date;cat **/home/ubuntu/user.txt**


![[Pasted image 20260923150340.png]]

en la parte infiero  vemos la bandera 
