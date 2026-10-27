# Checkmate

Máquina de nivel principiante en TryHackMe. El laboratorio se centra en detectar y explotar **credenciales débiles** en un formulario web.

## 1. Configuración del hosts

Para trabajar con el nombre del equipo, podemos agregar la entrada al archivo `/etc/hosts` de dos formas:

Con una sola línea desde la terminal:

```
echo 'MACHINE_IP nombre-del-equipo' >> /etc/hosts
```

O abriendo el archivo directamente:

```
nano /etc/hosts
```

## 2. Generación de diccionario con CeWL

En este laboratorio vamos a atacar el login con credenciales débiles. Para ello usamos **CeWL** (*Custom Word List generator*), una herramienta que crea una lista de palabras personalizada a partir del contenido de una página web:

```
cewl -d 2 -m 7 --lowercase --with-numbers -w cewl_words.txt http://jobs.thm:5002
```

Parámetros utilizados:

- `-d 2` → profundidad de rastreo de 2 enlaces.
- `-m 7` → longitud mínima de 7 caracteres.
- `--lowercase` → convierte todo a minúsculas.
- `--with-numbers` → incluye palabras que contienen números.
- `-w cewl_words.txt` → archivo de salida con el diccionario generado.

> Pendiente de documentar la fuerza bruta sobre el formulario y el resto de la máquina.
