
vamos a realizar  el comando de nmap 
nmap  -sV -sC 192.168.100.153 

![[Captura de pantalla 2026-09-27 235314.png]]

y como vemos ftp esta como anonymus  sin credeciales 
asi como vemos la imagen 

ftp anonymus@192.168.100.153

![[Pasted image 20260928000450.png]]

utilizamos la siguiente herramienta para enumera pagina web y utilizamos le siguiente comando 

gobuster dir -u http://192.168.100.153 -w /home/kali/SecLists/Discovery/Web-Content/common.txt

![[Pasted image 20260928001850.png]]


![[Pasted image 20260928002145.png]]

y encontramos los siguiente  password : 

![[Pasted image 20260928002214.png]]

![[Pasted image 20260928004326.png]]

y encontramos los siguiente usuario  Hulk y vemos que no temas contraseña 

![[Pasted image 20260928004301.png]]


![[Pasted image 20260928011234.png]]

![[Pasted image 20260928011413.png]]

![[Pasted image 20260928011435.png]]


User: hulk
password: fuerzabrutaxxx
![[Pasted image 20260928011507.png]]




