# EVM Wallet Cracker / Hunter

Herramienta de fuerza bruta acelerada con CUDA para recuperar seed phrases
BIP-39 perdidas u olvidadas de wallets compatibles con EVM.

<p align="center">
  <a href="https://github.com/disruptorh/Seed-Crack/releases/latest/download/seed-cracker">
    <img alt="Descargar" src="https://img.shields.io/badge/%E2%AC%87%20Download-latest%20release-2f6feb?style=for-the-badge&logo=github&logoColor=white">
  </a>
  <a href="https://github.com/disruptorh/Seed-Crack/releases/latest">
    <img alt="Versiones" src="https://img.shields.io/github/v/release/disruptorh/Seed-Crack?label=release&style=flat&logo=github&logoColor=white">
  </a>
  <a href="./LICENSE">
    <img alt="Licencia" src="https://img.shields.io/badge/licencia-Apache--2.0-blue?style=flat">
  </a>
</p>

---

## ⚠️ Aviso legal

**Esta herramienta es solo para uso educativo y forense.** Usarla contra wallets
que no son tuyas es ilegal. La probabilidad de encontrar por azar una wallet
activa es efectivamente cero: el espacio de búsqueda es de 2^128 a 2^256
dependientes de las palabras desconocidas, y no existe atajo conocido para el
hash Keccak-256.

---

## 📥 Descarga rápida

El asset es un ejecutable Linux x86-64 **sin extensión**, empaquetado con
PyInstaller (`onefile`). Incluye dentro el binario CUDA `wallet-cracker-bin` y la
wordlist `bip39.txt`, así que no necesitas ni Python ni PyInstaller para
usarlo, pero **sí una GPU NVIDIA con driver**, porque el motor llama a la GPU
(CUDA).

```bash
# 1. Descargar el ejecutable de la última release
curl -L -o seed-cracker https://github.com/disruptorh/Seed-Crack/releases/latest/download/seed-cracker

# 2. Dar permiso de ejecución y arrancar
chmod +x seed-cracker
./seed-cracker
```

En la máquina destino hace falta un X11/Wayland (la ventana es Tkinter) y las
librerías de OpenSSL 3 (`libcrypto.so.3`) y libstdc++, que el bundle no incluye.

---

## ⚠️ Lo que la herramienta **no** hace

Corrige las expectativas antes de usarla:

- **No es un cracker de claves privadas.** No ataca la clave del wallet ni
  ECDSA: prueba frases BIP-39 **completas y aleatorias** y compara la dirección
  EVM derivada.
- **El modo "100% GPU" es una etiqueta engañosa.** En el código, la generación
  de frases, PBKDF2-HMAC-SHA512, el BIP-32 y la derivación BIP-44 ocurren
  **todos en CPU**. Lo único que corre en la GPU es el kernel
  `compute_addresses_kernel`, que hace la multiplicación escalar de secp256k1 y
  el Keccak-256 de las claves ya derivadas.
- **El modo "híbrido" no existe como tal.** La GUI tiene un interruptor
  Híbrido/GPU, pero decide por existencia de ficheros, no por rendimiento: si
  encuentra `wallet-cracker-bin` junto al script usa ese (le llama "híbrido") y
  recorta el tercer argumento a 1–64; si no, busca `wallet-cracking-gpu/
  wallet-cracker-gpu` (o `wallet-cracker`). Ambas ramas ejecutan el mismo motor.
- **El binario precompilado no sustituye a la GPU.** `wallet-cracker-bin` está
  versionado para ahorrarte el CUDA Toolkit y el `nvcc`, pero el programa llama a
  `cudaMalloc` y al kernel: sin GPU o sin driver, `CUDA_CHECK` imprime
  `CUDA error ...` y hace `exit(1)`.
- **No hay `Makefile`.** Hay que compilar el `.cu` a mano.

En la práctica, la aceleración real es modesta: para derivar frases BIP-39 el
cuello de botella es PBKDF2, no la multiplicación de la curva.

---

## 🚀 Características

- **Frases parciales:** usa `?` como marcador para palabras desconocidas, p. ej.
  `abandon ? ability`. Las posiciones fijas se respetan; el resto se genera por
  fuerza bruta sobre las 2048 palabras de la wordlist.
- **Búsqueda 100 % aleatoria:** si no indicas frase parcial, genera frases de 12
  palabras aleatorias.
- **Checksum válido siempre:** tanto en modo aleatorio como en modo frase
  parcial, recalcula el checksum BIP-39 (SHA-256 de la entropía) y solo acepta
  candidatos que cuadran. En frase parcial eso deja una esperanza de ~1/16
  (12 palabras) o ~1/256 (24 palabras) por intento.
- **Lista de objetivos:** un `.txt` con una dirección EVM por línea (con o sin
  prefijo `0x`). Compara los 20 bytes de la dirección en crudo, sin pasar por
  EIP-55.
- **Progreso en vivo:** el binario escribe `Progreso: ...` en `stderr` cada
  segundo y la GUI lo muestra en la barra de estado; el match llega por `stdout`
  como `MATCH_FOUND:<target>:<frase>`.
- **Paralelismo por hilos CPU:** `num_threads` hilos, cada uno procesa lotes de
  512 frases, coordinados con `std::atomic` + `std::mutex` (`found`,
  `total_attempts`).

---

## 📦 Contenido

```text
wallet-cracker.cu     # motor CUDA (1205 líneas)
wallet-cracker.py     # GUI Tkinter que lanza el binario vía subprocess
wallet-cracker-bin    # binario precompilado (ELF x86-64, ~1,9 MB), versionado
bip39.txt             # wordlist BIP-39 en inglés (2048 palabras)
seed-cracker.spec     # PyInstaller: GUI + binario + wordlist → seed-cracker
```

### Sobre el desajuste de nombres

Es real y está en el repositorio, no es una errata:

- Los **fuentes** se llaman `wallet-cracker.cu` y `wallet-cracker.py`, y el
  binario que compilan es `wallet-cracker-bin`.
- El **spec de PyInstaller** y el **asset publicado** se llaman `seed-cracker`
  (sin `wallet-`).

`seed-cracker.spec` es lo que une ambos: empaqueta `wallet-cracker.py` con
`binaries=[('wallet-cracker-bin', '.')]` y `datas=[('bip39.txt', '.')]`, y nombra
la salida `seed-cracker`. Es decir, un único fichero ejecutable que lleva dentro
la GUI, el motor CUDA y la wordlist.

```bash
# Reconstruir el ejecutable de un solo fichero (solo si quieres tocarlo)
pip install pyinstaller && pyinstaller --clean seed-cracker.spec
```

Ojo: `seed-cracker.spec` **está ignorado por git** (`.gitignore` incluye
`*.spec`), así que existe en el árbol de trabajo pero no está versionado.

---

## 🚀 Uso rápido

### GUI (Tkinter)

```bash
# 1. Lanzar la GUI (usa el binario y la wordlist de junto al script)
python3 wallet-cracker.py
```

1. **Lista de objetivos:** botón *Seleccionar* para elegir el `.txt` con las
   direcciones EVM objetivo (una por línea, con o sin prefijo `0x`).
2. **Frase parcial (opcional):** escribe las palabras que recuerdas usando `?`
   para las que no. La GUI autocompleta con `?` hasta 12 palabras si escribes 12
   o menos, o hasta 24 si escribes más; más de 24 palabras es un error.
3. **Paralelismo:** número de hilos o batch (por defecto `65536` en el campo,
   recortado a 1–64 cuando está en modo híbrido).
4. **Iniciar Búsqueda** / **Detener**.

La GUI busca el motor en este orden: `wallet-cracker-bin` junto al script; si no
existe, `wallet-cracking-gpu/wallet-cracker-gpu` y luego
`wallet-cracking-gpu/wallet-cracker`. Con el layout por defecto
(`wallet-cracker-bin` versionado) siempre cae en el primero.

Un fichero de objetivos de ejemplo:

```bash
# Crear un objetivo válido (dirección pública, 40 hex, con o sin 0x)
printf '0x0000000000000000000000000000000000000000\n' > targets.txt
```

### CLI

```bash
# 1. Firma: <targets_file.txt> <bip39.txt> <num_threads> [frase_parcial]
./wallet-cracker-bin targets.txt bip39.txt 8
```

Es la firma real de `main()` en `wallet-cracker.cu:1078`. Las tres primeras son
obligatorias (con menos de 4 argumentos imprime el uso y devuelve 1); la frase
parcial es el cuarto argumento opcional. `bip39.txt` debe tener exactamente 2048
entradas, o el programa aborta.

```bash
# 2. Búsqueda con frase parcial (entrecomillado para que ? no la interprete el shell)
./wallet-cracker-bin targets.txt bip39.txt 8 "abandon ? ability ? ? ? ? ? ? ? ? ?"
```

```bash
# 3. Solo el progreso, que va por stderr
./wallet-cracker-bin targets.txt bip39.txt 8 2> progreso.log
```

Ejemplo completo, tal cual se pega:

```bash
# Objetivo imposible: la dirección cero no corresponde a ninguna frase alcanzable
printf '0x0000000000000000000000000000000000000000\n' > targets.txt

# Lanzar la búsqueda y ver el progreso durante unos segundos
timeout 10 ./wallet-cracker-bin targets.txt bip39.txt 8 2>&1 | tail -5
```

Códigos de salida: `0` si se encontró un match, `1` en cualquier error o si el
proceso se detiene sin match.

---

## 📦 Compilar desde código

### Requisitos

- **GPU NVIDIA** con driver y CUDA disponible. El motor no tiene modo CPU.
- **CUDA Toolkit** con `nvcc` en el `PATH` — solo para recompilar el `.cu`.
- **OpenSSL** (cabeceras para compilar, `libcrypto.so.3` en runtime).
- **Python 3.x con `tkinter`** — solo para la GUI desde código; el ejecutable de
  la release no lo necesita.

### Clonar

```bash
# 1. Clonar el repositorio
git clone https://github.com/disruptorh/Seed-Crack.git
cd Seed-Crack
```

### Compilar

**No hay `make` ni script de build.** El `.cu` se compila a mano:

```bash
# 2. Compilar el motor CUDA
nvcc -O3 -arch=sm_60 wallet-cracker.cu -o wallet-cracker-bin -lcrypto
```

Flags y por qué están ahí:

- `-O3`: optimización.
- `-arch=sm_60`: floor de Compute Capability 6.0. **Ajústalo a tu GPU**: `sm_61`,
  `sm_75`, `sm_86`, …
- `-lcrypto`: obligatorio. El `.cu` incluye `<openssl/evp.h>`,
  `<openssl/hmac.h>`, `<openssl/bn.h>`, `<openssl/ec.h>` y `<openssl/obj_mac.h>`,
  y de ahí salen PBKDF2-HMAC-SHA512 (BIP-39), HMAC-SHA512 (BIP-32), SHA-256
  (checksum BIP-39) y la aritmética de la curva. Sin ese flag falla el
  enlazado.

Para elegir `-arch` sin adivinar:

```bash
# Ver la Compute Capability de tu GPU y usarla como su_<major><minor>
nvidia-smi --query-gpu=name,compute_cap --format=csv
```

El nombre de salida importa: la GUI busca el binario en `wallet-cracker-bin`.

### Ejecutar los tests

```bash
# No hay tests automatizados en el repositorio
```

No hay suite de tests. La única verificación posible es manual: lanzar una
búsqueda corta y comprobar que el binario arranca, imprime
`Objetivos cargados: N` y `Progreso: ...` y que la GUI refleja ese progreso.

### Ejecutar la aplicación

```bash
# Motor directamente
./wallet-cracker-bin targets.txt bip39.txt 8

# GUI (usa wallet-cracker-bin y bip39.txt de junto al script)
python3 wallet-cracker.py
```

---

## 🧰 Comandos útiles / Opciones

### CLI

| Argumento | Obligatorio | Qué hace |
|---|---|---|
| `targets_file.txt` | sí | Una dirección EVM por línea. Prefijo `0x`/`0X` opcional. Las líneas no válidas se avisan y se ignoran. |
| `bip39.txt` | sí | Wordlist BIP-39. Debe tener **exactamente 2048 entradas** o aborta. |
| `num_threads` | sí | Número de hilos `std::thread` en paralelo. |
| `frase_parcial` | no (4º argumento) | Frase con `?` en las posiciones desconocidas. Máximo 24 palabras. Sin ella: fuerza bruta aleatoria de 12 palabras. |

Longitud de la frase: el binador decide el tamaño por lo que escribas
(`i <= 12` → frase de 12 palabras, si no de 24), no por el checksum.

Mensajes del propio programa:

```text
Uso: ./wallet-cracker-bin <targets_file.txt> <bip39.txt> <num_threads> [frase_parcial]
Error: no se pudo abrir el archivo de objetivos <fichero>
Error: No se encontró ninguna dirección objetivo válida en el archivo.
Error: archivo de palabras no contiene 2048 entradas
Palabra desconocida en frase parcial: <palabra>
Objetivos cargados: <n>
Hilos: <n>
Progreso: <n> intentos | Velocidad: <n> H/s
MATCH_FOUND:<direccion objetivo>:<frase encontrada>
```

El progreso va por `stderr` y el resultado por `stdout`, así que puedes separar
ambos:

```bash
# Guardar el progreso aparte y ver solo el resultado en pantalla
./wallet-cracker-bin targets.txt bip39.txt 8 2> progreso.log | tee match.txt
```

---

## 🗂️ Estructura del proyecto

```text
.
├── LICENSE                     # Apache-2.0
├── bip39.txt                   # wordlist BIP-39 en inglés (2048 palabras)
├── wallet-cracker.cu           # motor CUDA (1205 líneas)
├── wallet-cracker.py           # GUI Tkinter que lanza el binario vía subprocess
├── wallet-cracker-bin          # binario precompilado, versionado a propósito
├── seed-cracker.spec           # PyInstaller → ejecutable de un fichero (ignorado por git)
├── build/                      # salida de PyInstaller (ignorado por git)
└── dist/                       # ejecutables generados (ignorado por git)
```

---

## ⚙️ Detalles técnicos

- **BIP-39:** mnemónico → seed con PBKDF2-HMAC-SHA512, salt `"mnemonic"`. El
  checksum se recalcula en ambos modos con SHA-256 de la entropía y se aceptan
  solo candidatos válidos.
- **BIP-32:** HMAC-SHA512 con `"Bitcoin seed"` para la clave maestra y con la
  chain code como clave en cada derivación.
- **BIP-44:** derivación fija en `m/44'/60'/0'/0` (3 *hardened*: `0x80000000 | 44`,
  `0x80000000 | 60`, `0x80000000 | 0`, y una normal), con índice de dirección
  fijo en `0`.
- **secp256k1:** parámetros precalculados de la curva y multiplicación escalar en
  punto Jacobiano, con paso a afín.
- **Keccak-256:** hash de la pubkey sin comprimir `x || y`; la dirección son los
  20 últimos bytes.
- **Passphrase BIP-39:** vacía siempre (`bip39_mnemonic_to_seed(phrase, "", seed)`).
  Si tu wallet se creó con passphrase, esta herramienta **no** la encontrará.
- **Concurrencia:** `BATCH_SIZE = 512` frases por iteración e hilo, con la GPU
  reservada una sola vez; `found` y `total_attempts` son `std::atomic` protegidos
  por `std::mutex`.

---

## 🔗 Relacionado

- [disruptorh/Direction-Generation-EVM](https://github.com/disruptorh/Direction-Generation-EVM) —
  el generador de direcciones EVM complementario (mismo motor CUDA, mismo BIP-44).

---

## 🔐 Seguridad

- **Herramienta offline por diseño.** Para máxima seguridad, úsala en una máquina
  sin red.
- Los ficheros de objetivos solo contienen direcciones públicas, pero son datos
  que vinculan una identidad con un saldo. Trátalos como información sensible.
- **Passphrase BIP-39 no soportada:** si la frase que buscas se creó con una
  passphrase, la herramienta no la va a encontrar. El passphrase se asume vacío.
- Si encuentras un match, la frase que se imprime por pantalla es una wallet
  con fondos. Muévela de inmediato o, al menos, no la reutilices ni la
  compartas.

---

## 📄 Licencia

Apache-2.0 — ver [LICENSE](LICENSE).
