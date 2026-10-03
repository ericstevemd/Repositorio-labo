
Como primer paso podemos realizar el reconocimiento  de puerto con nmap 
y ejecutamos el siguiente comando 

nmap -sV -sC 172.17.0.2

![[Pasted image 20261001223106.png]]

vemos que tenemos los siguiente puerto para ejecutar 

el puerto 22  y puerto 12345
y utilizamos la herramienta de gobusters 
para buscar posible usuarios 
![[Pasted image 20261001223311.png]]

asi mismos revisamos la enumeracion de numero 

![[Pasted image 20261001224832.png]]

Realizamos enumeracion  

#!/bin/bash

for i in {1..25}; do
    curl -s -X GET "http://172.17.0.2:12345/user/$i" | grep username | cut -d\" -f4 >> usernamess.txt
done;


solo sacamos los usuario 

![[Pasted image 20261001234611.png]]


![[Pasted image 20261001235656.png]]

![[Pasted image 20261002000057.png]]


![[Pasted image 20261002000555.png]]

![[Pasted image 20261002000629.png]]

![[Pasted image 20261002000820.png]]

----+-----------+----------------------------------+

| id | usuario   | password                         |
+----+-----------+----------------------------------+
|  1 | alice     | 8bdffaa69d328c1d4ae3aeadc97de223 |
|  2 | bob       | d8578edf8458ce06fbc5bb76a58c5ca4 |
|  3 | charlie   | e99a18c428cb38d5f260853678922e03 |
|  4 | 
| aa87ddc5b4c24406d26ddad771ef44b0 |
|  5 | diana     | e10adc3949ba59abbe56e057f20f883e 


![[Pasted image 20261002001347.png]]

![[Pasted image 20261002002022.png]]

![[Pasted image 20261002002500.png]]


![[Pasted image 20261002004836.png]]


![[Pasted image 20261002010403.png]]