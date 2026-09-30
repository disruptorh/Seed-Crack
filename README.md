# EVM Wallet Cracker / Hunter

Herramienta de fuerza bruta acelerada con CUDA para recuperar seed phrases
BIP-39 perdidas o olvidadas de wallets compatibles con EVM.

![Licencia](https://img.shields.io/badge/License-Apache--2.0-yellow.svg)

## ⚠️ Aviso legal

**Esta herramienta es solo para uso educativo y forense.** Usarla contra
wallets que no son tuyas es ilegal. La probabilidad de encontrar por azar una
wallet activa es efectivamente cero: el espacio de búsqueda es de 2^128 a 2^256
dependientes de las palabras desconocidas, y no existe atajo conocido para el
hash Keccak-256.

## ⚠️ Lo que la herramienta **no** hace

Corrige las expectativas antes de usarla:

- **No es un cracker de claves privadas.** No ataca el clave del wallet ni
  ECDSA: prueba frases BIP-39 **completas y aleatorias** y compara la dirección
  EVM derivada.
- **El modo "100% GPU" es una etiqueta engañosa.** En el código, la generación
  de frases, PBKDF2-HMAC-SHA512, el BIP-32 y la derivación BIP-44 ocurren
  **todos en CPU**. Lo único que corre en la GPU es el kernel
  `compute_addresses_kernel`, que hace la mult. escalar de secp256k1 y el
  Keccak-256 de las claves ya derivadas.
- **El modo "híbrido" no existe como tal.** La GUI tiene un interruptor
  Híbrido/GPU, pero ambas ramas lanzan el **mismo binario**: lo único que cambia
  es el tercer argumento (en híbrido se recorta a 1–64).
- **No hay `Makefile`.** Hay que compilar el `.cu` a mano.

En la práctica, la aceleración real es modesta: para deriving frases BIP-39
el cuello de botella es PBKDF2 (600k+ iteraciones por frase), no la
multiplicación de la curva.

## 🚀 Características

- **Frases parciales:** usa `?` como marcador para palabras desconocidas, p. ej.
  `abandon ? ability ? ? ?`. Las posiciones fijas se respetan; el resto se
  busca por fuerza bruta sobre las 2048 palabras.
- **Búsqueda 100 % aleatoria:** si no indicas frase parcial, genera frases de
  12 palabras aleatorias.
- **Lista de objetivos:** un `.txt` con una dirección EVM por línea. Compara los
  20 bytes de la dirección en crudo (`memcmp`), sin pasar por EIP-55.
- **Progreso en vivo:** el binario escribe `Progreso: ...` en `stderr` y la GUI
  lo muestra en la barra de estado; el match llega por `stdout` como
  `MATCH_FOUND:<target>:<frase>`.
- **Paralelismo por hilos CPU:** cada hilo procesa lotes de 512 frases y
  coordina con `std::atomic` + `std::mutex` (`found`, `total_attempts`).

## 📦 Contenido

```
wallet-cracker.cu     # motor CUDA (1123 líneas)
wallet-cracker.py     # GUI Tkinter que lanza el binario
wallet-cracker-bin    # binario precompilado (ELF x86-64, 1,9 MB) versionado
bip39.txt             # wordlist BIP-39 en inglés (2048 palabras)
```

El binario precompilado está versionado a propósito, para poder usar la
herramienta sin tarjeta NVIDIA ni toolchain CUDA.

## 🛠️ Requisitos

- **Python 3.x** con `tkinter` — solo para la GUI; el binario no necesita nada
- **GPU NVIDIA + CUDA Toolkit (con `nvcc`)** — solo para recompilar
- El binario precompilado ya incluye las bibliotecas CUDA, pero necesita que
  estén presentes en el sistema.

## 🔨 Compilación

No hay `make`. Compila a mano:

```bash
nvcc -O3 -arch=sm_60 wallet-cracker.cu -o wallet-cracker-bin -lcrypto
```

Ajusta `-arch` a tu GPU (`sm_61`, `sm_75`, `sm_86`, …). `-lcrypto` es
necesario para PBKDF2/HMAC de OpenSSL.

## 🖥 Uso

### GUI (Tkinter)

```bash
python3 wallet-cracker.py
```

1. **Lista de objetivos:** botón *Seleccionar* para elegir el `.txt` con las
   direcciones EVM objetivo (una por línea, con o sin prefijo `0x`).
2. **Frase parcial (opcional):** escribe las palabras que recuerdas usando `?`
   para las que no. El binario infiere si es de 12 o 24 palabras por la
   cantidad que escribas.
3. **Paralelismo:** número de hilos o batch.
4. **Iniciar Búsqueda** / **Detener**.

### CLI

```bash
./wallet-cracker-bin <targets.txt> <bip39.txt> <num_threads> [frase_parcial]
```

```bash
./wallet-cracker-bin targets.txt bip39.txt 8 "abandon ? ability"
```

- Las 3 primeras son obligatorias; la frase parcial es opcional.
- `bip39.txt` debe estar accesible (el GUI lo busca junto al script).

## ⚙️ Detalles técnicos

- **BIP-39:** mnemónico → seed con PBKDF2-HMAC-SHA512 (2048 iteraciones,
  salt `"mnemonic"`), y **el checksum se recalcula** antes de aceptar cada
  frase candidata: en modo frase parcial genera índices aleatorios y descarta
  los que no cuadran (esperanza ≈ 1/16 para 12 palabras, 1/256 para 24).
- **BIP-32:** HMAC-SHA512 con `"Bitcoin seed"` y con la chain code como clave.
- **BIP-44:** derivación fija en `m/44'/60'/0'/0/0` (3 *hardened* + 1 normal).
- **secp256k1:** multiplicación escalar en punto Jacobiano, con paso a afín.
- **Keccak-256:** hash de la pubkey sin comprimir `x || y`; la dirección son los
  20 últimos bytes.
- **Passphrase BIP-39:** vacía siempre (`bip39_mnemonic_to_seed(phrase, "", seed)`).

## 🔗 Relationado

- [Direction-Generation-EVM](../Direction-Generation-EVM/README.md) — el
  generador de direcciones EVM complementario (mismo motor CUDA, mismo BIP-44).
- [seed-tools](../seed-tools/README.md) — lanzadores de escritorio. ⚠️ El de
  esta herramienta apunta a `Seed-Cracking/dist/seed-cracker`, ruta que ya no
  existe.

## Licencia

Apache-2.0 — ver [LICENSE](LICENSE).
