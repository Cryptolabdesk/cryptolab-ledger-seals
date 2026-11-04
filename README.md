# cryptolab-ledger-seals

Sellos diarios encadenados del registro de lecturas de CryptoLab, anclados en Bitcoin con
[OpenTimestamps](https://opentimestamps.org).

Sirve para que cualquiera pueda comprobar dos cosas sin tener acceso a nuestros datos:

1. Que el registro de un día **existía tal cual en una fecha**, porque el `.ots` ancla el hash del
   sello en un bloque de Bitcoin.
2. Que **ningún día anterior se ha reescrito**, porque cada sello incluye el hash del anterior.

## Qué contiene y qué NO contiene un sello

**Contiene solo hashes, recuentos y fechas:**

| Campo | Contenido |
|---|---|
| `seal_date` | día UTC sellado (lo capturado ese día) |
| `sealed_at` | cuándo se calculó el sello (siempre después de terminar el día) |
| `judge_rows` | número de veredictos del Judge capturados ese día |
| `judge_digest` | hash de las filas del Judge del día. Cubre el **texto completo** de cada veredicto, aunque el texto **no** está aquí |
| `layer_rows` | número de lecturas capturadas por capa (`technical`, `onchain`, `macro`) |
| `layer_digest` | hash de las lecturas de capa del día |
| `prev_seal_hash` | `seal_hash` del sello anterior (`null` en el primero) |
| `seal_hash` | hash del propio sello |
| `algoritmo` | referencia a esta descripción |

**NO contiene:**

- ningún texto de los briefs;
- ninguna lectura (alcista, bajista, rango…);
- ningún precio;
- ninguna serie ni dato de ningún proveedor;
- ningún dato personal.

Antes de escribir un sello, el código comprueba una lista blanca de campos y tipos. Si aparece
cualquier otra cosa, el sello no se escribe ni se publica.

El repositorio contiene **solo** `seals/AAAA-MM-DD.json`, `seals/AAAA-MM-DD.json.ots` y este
README.

## Cómo verificar

Ejemplo con el sello del **03/11/2026**.

### 1. Descargar el sello y su prueba

```bash
B=https://raw.githubusercontent.com/Cryptolabdesk/cryptolab-ledger-seals/main
curl -sLO "$B/seals/2026-11-03.json"
curl -sLO "$B/seals/2026-11-03.json.ots"
```

### 2. Recalcular el hash del fichero

```bash
sha256sum 2026-11-03.json
# db32f84f329a60ec7bb9c65cefd48d041e173d9d564d42daef7cfb400118ed7f  2026-11-03.json
```

En Windows: `certutil -hashfile 2026-11-03.json SHA256`.

Ese hash tiene que ser el mismo que lleva la prueba dentro:

```bash
pip install opentimestamps-client
ots info 2026-11-03.json.ots | head -1
# File sha256 hash: db32f84f329a60ec7bb9c65cefd48d041e173d9d564d42daef7cfb400118ed7f
```

### 3. Verificar el anclaje en Bitcoin

**Con un nodo de Bitcoin propio** (`bitcoind` en local, o con `--bitcoin-node`):

```bash
ots verify 2026-11-03.json.ots
# Success! Bitcoin block <altura> attests existence as of <fecha>
```

**Sin nodo propio**:

```bash
ots --no-bitcoin verify 2026-11-03.json.ots   # muestra la altura del bloque que la prueba dice usar
ots info 2026-11-03.json.ots                   # al final: la raíz de Merkle esperada en ese bloque
```

Después hay que comparar esa raíz con la del bloque de la misma altura en un explorador público
independiente; tienen que coincidir. Por ejemplo:

```bash
curl -s https://blockstream.info/api/block-height/<altura>
curl -s https://blockstream.info/api/block/<hash>
```

El anclaje en Bitcoin tarda unas horas desde que se crea el sello. Mientras no llega, `ots verify`
dice `Pending confirmation`, y la prueba de este repositorio se actualiza en la ejecución de la
noche siguiente.

### 4. Recalcular `seal_hash` y comprobar la cadena

`H(p1, …, pn)` es el SHA-256 de cada parte en UTF-8 seguida del byte `0x1f`.

```bash
python3 - <<'EOF'
import json, hashlib
d = json.load(open("2026-11-03.json"))
h = hashlib.sha256()
for p in (d["seal_date"], d["judge_digest"], d["layer_digest"], d["prev_seal_hash"] or ""):
    h.update(p.encode()); h.update(b"\x1f")
print(h.hexdigest() == d["seal_hash"], d["seal_hash"])
EOF
# True d9627669505e65b0b50bbb6583d2a3b5585fa019f6f85501ca03ccb27dba2043
```

Para la cadena, el `prev_seal_hash` de cada día tiene que ser igual al `seal_hash` del día
anterior.

## Cómo se calculan los digests

Los objetos JSON se serializan con claves ordenadas, sin espacios y con `ensure_ascii=false`.

| Hash | Cálculo |
|---|---|
| `judge_digest` | `H(f₁, …, f_k)` con `f_i = "<row_hash_i>:<H(brief_i)>"`, para las filas del Judge capturadas ese día, ordenadas por hora de captura. `row_hash_i` encadena cada fila con la anterior en el registro; `H(brief_i)` cubre el texto íntegro del veredicto |
| `layer_digest` | igual, con `"<row_hash_i>:<payload_sha256_i>"` de las lecturas de capa |
| `seal_hash` | `H(seal_date, judge_digest, layer_digest, prev_seal_hash or "")` |

Las filas de origen están en un registro **append-only**: la base de datos rechaza UPDATE y DELETE.
Pueden mostrarse a un auditor para que recalcule `judge_digest` y `layer_digest`. Sin esas filas
se pueden comprobar los pasos 2 a 4, pero no los digests.
