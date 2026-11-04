# cryptolab-ledger-seals

Sellos **horarios** (desde el 04/11/2026) y **diarios** encadenados del registro de lecturas de
CryptoLab, anclados en Bitcoin con [OpenTimestamps](https://opentimestamps.org).

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

El repositorio contiene **solo** estos ficheros:

- `seals/AAAA-MM-DD.json` y su `.ots`: diarios;
- `seals/horas/AAAA-MM-DD/HH.json` y su `.ots`: horarios;
- este README.

## Sellos horarios (desde 2026-11-04 00:00 UTC)

Cada hora UTC se sella lo **capturado** en `[HH:00, HH+1:00)`, unos 7 minutos después de terminar
la hora. **También las horas sin lecturas**, de modo que un hueco en la secuencia solo puede ser un
fallo y es visible.

Campos de un sello horario:

| Campo | Contenido |
|---|---|
| `seal_hour` | inicio de la hora sellada (UTC) |
| `sealed_at` | cuándo se calculó |
| `judge_rows`, `layer_rows` | recuentos |
| `judge_digest`, `layer_digest` | igual que en el diario, pero sobre esa hora. En una hora vacía valen `H("vacio", seal_hour)` |
| `prev_hourly_seal_hash` | sello horario anterior |
| `seal_hash` | `H("h1", seal_hour, judge_digest, layer_digest, prev_hourly_seal_hash or "")` |
| `algoritmo` | `v2-horario…` |

**Diario v2** (desde el primer día completo de sellos horarios). Añade `hourly_count = 24` y
`hourly_digest = H(seal_hash de las horas 00…23)`, y
`seal_hash = H("d2", seal_date, judge_digest, layer_digest, hourly_digest, prev_seal_hash or "")`.
Su `prev_seal_hash` sigue la misma cadena diaria: el primer diario v2 apunta al último v1.

**Las horas del 04/11 anteriores a la activación (00:00–05:00) se sellaron con retraso**, a las
06:04 UTC. Por la regla de abajo, las lecturas de esas horas no cuentan como ancladas antes de su
resultado a 4 h.

## Regla «anclada antes del resultado» (`cl-anchor-before-outcome-v1`)

Una lectura cuenta como **verificable externamente** para un horizonte h solo si cumple
`T_ancla + 2 h ≤ G + h`, donde:

- `G` es la hora de la lectura;
- `T_ancla` es el `time` del bloque de Bitcoin de **menor altura** que ancla el sello **horario**
  que la contiene;
- **2 h** es el margen: Bitcoin acepta cabeceras con hasta 2 h de adelanto.

Las demás quedan en NO, PENDIENTE (todavía sin anclaje) o NO_APLICA (sin sello horario), y nunca
se suman a las verificables. Texto completo del fichero de la regla (sha256
`d6be9196bd46bb51dc27732e3c2e44bc1211a612188d9b1e4a1ff092b444a8e9`, anclado con OpenTimestamps el
04/11):

<details><summary><code>regla_anclaje_v1.json</code></summary>

```json
{
  "rule_id": "cl-anchor-before-outcome-v1",
  "status": "SELLADA_ANTES_DE_IMPLEMENTAR_Y_DE_VER_ANCLAJES_HORARIOS",
  "unidad": "cada lectura canónica de research.layer_ledger_v1 (is_canonical_public = true) y cada veredicto de research.forecast_ledger_v1",
  "G": {"capas": "generated_at", "judge": "judge_generated_at"},
  "horizontes_h": {"technical": [4], "onchain": [24], "macro": [24], "judge": [24, 4]},
  "sello_que_la_contiene": "el sello HORARIO de la hora UTC [HH:00, HH+1:00) que contiene captured_at de la fila; la fila tiene que estar incluida en su digest (comprobable recalculando con las filas)",
  "momento_que_cuenta": {
    "T_ancla": "campo 'time' de la cabecera del bloque de Bitcoin de MENOR altura entre las atestaciones BitcoinBlockHeaderAttestation del .ots de ese sello horario",
    "validez_de_la_atestacion": "la raíz de Merkle que da la prueba debe coincidir con la 'merkleroot' de la cabecera de esa altura, según un nodo completo propio y al menos un explorador público independiente (blockstream.info o mempool.space)"
  },
  "margen": {
    "M_horas": 2,
    "motivo": "la regla de consenso de Bitcoin admite cabeceras con 'time' hasta 2 h por delante del tiempo ajustado de la red, y 'time' puede quedar por detrás del momento real de minado; se descuenta 2 h a favor de la duda"
  },
  "regla": "anclada_antes_del_resultado(h) = SI  si  T_ancla + M <= G + h ;  NO  si T_ancla + M > G + h ;  PENDIENTE si el .ots aún no tiene atestación de Bitcoin (se resuelve en cuanto llegue: si llega tarde, queda NO para siempre) ;  NO_APLICA si la fila no está en un sello horario (capturas anteriores a la activación de los sellos horarios o a fallos sin recuperar)",
  "uso_publico": "SOLO las lecturas con anclada_antes_del_resultado(h) = SI cuentan como «verificables externamente» para ese horizonte. El resto se muestra aparte (con su recuento) y nunca se suma",
  "descriptivo_no_cuenta": "se informa también la proporción con M = 0, solo como contexto",
  "inmutabilidad": "la evaluación de cada lectura se guarda (append-only) cuando su sello tiene atestación; un cambio de regla sería cl-anchor-before-outcome-v2 con fecha y sello propios, sin reevaluar en silencio"
}
```

</details>

## Cómo verificar

Ejemplo con el sello del **03/11/2026**. Hay tres formas de verificar el anclaje (A, B o C); D comprueba además la cadena:

- **A. En el navegador**, sin instalar nada: sirve en Windows, macOS, Linux o el móvil.
- **B. Con la línea de comandos** de OpenTimestamps.
- **C. Con un script de Python**, si la línea de comandos no arranca, por ejemplo en Windows.

### A. Verificación en el navegador (opentimestamps.org)

1. Descarga los dos ficheros desde GitHub: abre cada uno y pulsa *Download raw file*.
   - [`seals/2026-11-03.json`](seals/2026-11-03.json)
   - [`seals/2026-11-03.json.ots`](seals/2026-11-03.json.ots)
2. Abre <https://opentimestamps.org> y, en **Stamp & Verify**, arrastra el fichero
   **`2026-11-03.json.ots`** («an .ots proof file to verify»).
3. Cuando la página lo pida, arrastra el fichero original **`2026-11-03.json`**. El hash se calcula
   en tu navegador: el fichero no se sube a ningún sitio.
4. La página muestra el bloque de Bitcoin que lo atestigua y la fecha. Para este sello son los
   bloques **974 473** (2026-11-04 04:04:33 UTC) y **974 476** (04:08:09 UTC).

Si arrastras un JSON distinto, o cambias un solo carácter, la verificación falla.

**No abras ni guardes el JSON con un editor antes de verificarlo**: un salto de línea o un cambio de
codificación alteran el hash. Usa la descarga directa.

### B. Verificación con la línea de comandos

#### 1. Descargar el sello y su prueba

```bash
B=https://raw.githubusercontent.com/Cryptolabdesk/cryptolab-ledger-seals/main
curl -sLO "$B/seals/2026-11-03.json"
curl -sLO "$B/seals/2026-11-03.json.ots"
```

#### 2. Recalcular el hash del fichero

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

#### 3. Verificar el anclaje en Bitcoin

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

El anclaje en Bitcoin tarda desde unos minutos hasta unas horas desde que se crea el sello.
Mientras no llega, `ots verify` dice `Pending confirmation`, y la prueba de este repositorio se
actualiza en la siguiente ejecución.

### C. Alternativa: script de Python

Usa la librería `opentimestamps` (sin la CLI, que en Windows suele fallar por una librería nativa).
Hace tres cosas:

- comprueba el hash del fichero;
- completa la prueba consultando los calendarios;
- compara la raíz de Merkle con el bloque en dos exploradores independientes, blockstream.info y
  mempool.space.

```bash
pip install opentimestamps
python verificar_ots.py 2026-11-03.json 2026-11-03.json.ots
```

<details><summary><code>verificar_ots.py</code></summary>

```python
import hashlib, json, sys, urllib.request, datetime as dt
from opentimestamps.core.timestamp import DetachedTimestampFile
from opentimestamps.core.serialize import StreamDeserializationContext
from opentimestamps.core.notary import PendingAttestation, BitcoinBlockHeaderAttestation
from opentimestamps.calendar import RemoteCalendar

def walk(ts):
    yield ts
    for st in ts.ops.values():
        yield from walk(st)

def get(url):
    req = urllib.request.Request(url, headers={"User-Agent": "ledger-seals-verify"})
    with urllib.request.urlopen(req, timeout=20) as r:
        return r.read().decode()

fjson, fots = sys.argv[1], sys.argv[2]
data = open(fjson, "rb").read()
with open(fots, "rb") as f:
    dtf = DetachedTimestampFile.deserialize(StreamDeserializationContext(f))
print("hash del JSON coincide con el .ots:", hashlib.sha256(data).digest() == dtf.file_digest)
for st in walk(dtf.timestamp):                      # completa lo pendiente con los calendarios
    for att in list(st.attestations):
        if isinstance(att, PendingAttestation):
            try:
                st.merge(RemoteCalendar(att.uri).get_timestamp(st.msg))
            except Exception:
                pass
for msg, att in dtf.timestamp.all_attestations():
    if isinstance(att, BitcoinBlockHeaderAttestation):
        raiz = msg[::-1].hex()
        for base in ("https://blockstream.info/api", "https://mempool.space/api"):
            h = get(f"{base}/block-height/{att.height}").strip()
            b = json.loads(get(f"{base}/block/{h}"))
            hora = dt.datetime.fromtimestamp(b["timestamp"], dt.timezone.utc)
            print(f"bloque {att.height} ({hora:%Y-%m-%d %H:%M:%S} UTC) en {base.split('/')[2]}: "
                  f"{'COINCIDE' if b['merkle_root'] == raiz else 'NO COINCIDE'}")
```

</details>

Salida esperada con el sello del 03/11:

```
hash del JSON coincide con el .ots: True
bloque 974476 (2026-11-04 04:08:09 UTC) en blockstream.info: COINCIDE
bloque 974476 (2026-11-04 04:08:09 UTC) en mempool.space: COINCIDE
bloque 974473 (2026-11-04 04:04:33 UTC) en blockstream.info: COINCIDE
bloque 974473 (2026-11-04 04:04:33 UTC) en mempool.space: COINCIDE
```

### D. Recalcular `seal_hash` y comprobar la cadena

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

## Reglas anunciadas

| Regla | sha256 del fichero | Anclada | Qué decide |
|---|---|---|---|
| `cl-anchor-before-outcome-v1` | `d6be9196bd46bb51dc27732e3c2e44bc1211a612188d9b1e4a1ff092b444a8e9` | OpenTimestamps, 2026-11-04 ~05:22 UTC | Cuándo una lectura cuenta como **anclada antes de su resultado**: el bloque de Bitcoin que ancla su sello horario, más **2 h de margen**, tiene que ser anterior a la hora de la lectura más su horizonte. Se publicará el fichero completo junto con los sellos horarios |

## Registro de cambios

Este repositorio **no reescribe su historial**: ni force-push ni commits borrados. Los errores se
corrigen con commits nuevos y se anotan aquí.

| Fecha (UTC) | Commit | Cambio |
|---|---|---|
| 2026-11-04 03:42 | `e0453a5` | README inicial |
| 2026-11-04 03:43 | `c040301` | Primer sello (`2026-11-03`), con su `.ots` pendiente de Bitcoin |
| 2026-11-04 04:29 | `0a4cb14` | README: cómo verificar |
| 2026-11-04 04:33 | `b0ebfac` | `.ots` del 03/11 completado: bloques 974 473 y 974 476. **Error**: subió también `seals/2026-11-03.json.ots.bak`, la copia que `ots upgrade` deja del `.ots` pendiente anterior. No contenía datos, solo esa prueba antigua |
| 2026-11-04 05:12 | `1443236` | README: verificación en el navegador y script alternativo |
| 2026-11-04 05:13 | **`06f269b`** | **Retirado `seals/2026-11-03.json.ots.bak`.** Desde aquí la rutina solo publica `README.md`, `seals/*.json` y `seals/*.json.ots`. El `.bak` sigue visible en el historial de `b0ebfac`, **intencionadamente: no se reescribe el historial** |
| 2026-11-04 05:22 | `3760733` | Registro de cambios y regla `cl-anchor-before-outcome-v1` anunciada por su hash |
| 2026-11-04 06:04 | `6dee929` | **Primeros sellos horarios**: 04/11, de 00:00 a 05:00, sellados con retraso porque son anteriores a la activación |
| 2026-11-04 06:1x | (este) | README: sellos horarios, diario v2 y texto completo de la regla |
