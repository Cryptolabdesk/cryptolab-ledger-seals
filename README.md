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
- `adendas/AAAA-MM-DD_<nombre>.json` y su `.ots`: el **hash** de un documento interno (una adenda
  sellada **antes** del hecho que declara), con su fecha de sellado. Sin el texto del documento
  (ver la nota de abajo: hoy no están en `main`);
- este README.

## Adendas y cadena canónica

Las adendas y los criterios internos se encadenan en un registro propio, `SELLOS.txt`. Cada
bloque nuevo lleva el sha256 de todo lo anterior, de modo que reescribir un bloque viejo rompe la
cadena. Ese registro todavía no se publica (se publicará más adelante), pero el hash de cada adenda que
afecta a lo que se publica aquí va a `adendas/` y se ancla en Bitcoin con su `.ots`.

**Hash de `SELLOS.txt`.** Desde el 05/11/2026 el propio registro se ancla en Bitcoin. Versión anclada más
reciente: `824e2c9961a622776941359c7c2308dc365e094932e6274908423d7175394116` (38 148 B), en el bloque
**974 635** (2026-11-05 07:19:14 UTC). Su `.ots` se publicará junto con el registro.

| Adenda | sha256 del documento | Dónde está | Qué declara |
|---|---|---|---|
| P4-H, despliegue de C0 | `5f69062eff944e7738b5d567c877764146f197e1b72b9100e3a1be18d076e23e` (sellada 2026-11-04 15:35:52 UTC) | [`adendas/2026-11-04_p4h_despliegue_c0.json`](https://github.com/Cryptolabdesk/cryptolab-ledger-seals/blob/659ef24d0abebcea7e6bead93a14cb1fab43a4bb/adendas/2026-11-04_p4h_despliegue_c0.json) y su `.ots`, en el commit `659ef24` | El cambio de versión del Técnico del 05/11 se declara antes: no es una anomalía de la vigilancia P4-H |

**Nota 1 (04/11/2026): `adendas/` no está ahora en `main`.** La rutina automática de sellos
solo conserva `README.md` y `seals/`, y en su pasada de las 17:07 UTC (`c770e72`) retiró la adenda
publicada a las 16:29. **Sigue en el historial, que no se reescribe**: el enlace de arriba apunta
al commit fijo `659ef24`. La rutina se corregirá para conservar `adendas/` (cambio A0, previsto
para el 07/11), y entonces se repondrá el fichero.

**Nota 2 (04/11/2026): cadena canónica.** La copia de `SELLOS.txt` del servidor no tiene todavía
los bloques sellados desde las 15:35 UTC del 04/11: el fichero está protegido contra escritura y se
completará a mano. **Mientras tanto, la cadena canónica es la copia local de `SELLOS.txt` más
`adendas/` con sus `.ots`.** La copia del servidor es un prefijo exacto de la local (sha256
`397be018…`), y solo le faltan esos bloques. Esta nota se retira cuando las dos copias coincidan.

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

Las horas del 04/11 de 00:00 a 05:00 se sellaron todas a las 06:04 UTC. **Cada lectura se evalúa
con la regla de abajo, sin excepciones.**

**Ventana de exposición y hora tardía** (presentación, no criterio):

- Cada lectura se registra en los minutos siguientes, y su sello se publica aquí en la pasada horaria
  (a las :07 de la hora siguiente), con un máximo de ~67 minutos de ventana.
- La **ventana de exposición** de una lectura son los minutos entre su generación y su sello. Con la
  tarea horaria funcionando está acotada por diseño a ~67 min (60 + 7) y no tiene umbral.
- Una hora es **tardía** si su sello se calculó más de 15 min después de terminar. Es un indicador
  del funcionamiento de la tarea horaria, **no un criterio de validez**: la validez la decide solo la
  regla de anclaje. Las horas 00:00–04:00 del 04/11 son tardías (304 a 64 min) y no contenían
  ninguna lectura.

> **Corrección (05/11).** La frase publicada en `1f865dc` («las lecturas de esas horas no cuentan como
> ancladas antes de su resultado a 4 h») **era incorrecta**: la regla sellada
> `cl-anchor-before-outcome-v1` **nunca tuvo un corte por activación** y se aplica igual a toda
> lectura.
>
> Caso que lo muestra: la lectura técnica de las 05:01 UTC del 04/11.
> - Su hora se selló a las 06:04, 4 min después de terminar, así que no es tardía; su ventana de
>   exposición fue de 63 min.
> - Quedó anclada en Bitcoin a las 06:22.
> - **Cumple** la regla: 06:22 + 2 h = 08:22, antes de 05:01 + 4 h = 09:01.
>
> La regla no ha cambiado.

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

**Calcula siempre el hash sobre el fichero *raw* de GitHub** (*Download raw file*, o una URL de
`raw.githubusercontent.com`, mejor con el commit en lugar de `main`), **nunca sobre una copia hecha con
`git clone`**. Según tu configuración de git (`core.autocrlf`, activado por defecto en Git para Windows),
el checkout puede cambiar los saltos de línea, y entonces el hash no coincide aunque el fichero publicado
sea el correcto.

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
| Vigilancia P4-H2 (preregistro) | `18c7e0dc2c2194800643550012d72f804dbd98cb71c0ca17fc87b3b863b45726` | OpenTimestamps, 2026-11-04 18:40:28 UTC (antes de aplicar ningún cambio) | La primera vigilancia, P4-H, **no cumplió** su criterio C4, «repositorio igual al local en al menos 47 de 48 chequeos», por una incidencia de GitHub del 04/11. P4-H2 cuenta **7 días seguidos** desde la corrección. Un fallo de sincronía solo se perdona si coincide con una incidencia de GitHub documentada en el momento **y** todo se sube en orden en ≤ 2 h. Cualquier otro fallo reinicia la cuenta. Adenda 1 (sha256 `4282f45c65eb9d784a0fef13ee0126f781726fed7b6310fb496b70714d9e49d0`), sellada el 2026-11-05 a las 00:38:23 UTC; sus ficheros se publicarán con A0 (previsto el sábado 07/11) |

## Registro de proceso

> **Aviso:** los resultados históricos mostrados describen lecturas ya publicadas y su comparación con
> el precio posterior. No garantizan ni aseguran resultados futuros. Toda la información de esta sección
> es estrictamente informativa y no constituye asesoramiento ni oferta de inversión.

Lo que no salió bien en nuestras pruebas, contado aunque no nos favorezca. Es una copia **literal**
de un apartado de nuestra bitácora pública, `publico/BITACORA_PUBLICA.md`. Ese fichero es interno y
todavía no es público; su sha256 sirve para comprobar después que lo publicado no cambió:
`a6e135a1915a8ac721e3814df0242ac1bc7ae39b70b36b5c3d8eec63b7cad684`, sellado en nuestra cadena interna el 2026-11-05 a las 19:47:59 UTC. La versión anterior del
apartado (bitácora `5d19e28ad8763ab9f756f64ad68a6aa2b1d285242e40bb6e3a69584c72131774`, publicada en
`a6745f4`) queda en el historial. **La bitácora completa se publicará más adelante.**

- **La prueba de C0b v1.2 (adenda 2, 05/11/2026) falló, y esa versión no se despliega.** Se hizo en
  corridas de prueba, sin publicar nada. La revisó a ciegas un lector independiente (otro modelo), que
  no sabía qué texto venía de producción y cuál de la versión nueva.
  - **Antes, la versión 1.1 también falló la revisión independiente**, por un error de signo del flujo
    neto. Nuestra primera revisión, hecha por quien construyó la versión, no lo detectó.
  - **Signo del flujo neto con v1.2.** Tras el filtro, ningún texto de la versión nueva tuvo error de
    signo; el filtro quitó uno que había escrito el modelo. Sí lo tuvieron 3 de los 7 textos de
    producción usados como comparación, pero 2 de esos 3 casos se eligieron justamente por haber
    publicado ese error, así que no es una tasa.
  - **Falló en dos puntos.** Un texto decía que faltaban datos de la mempool. Estaban en el corte, pero
    ese agente no los recibía, por un defecto de construcción nuestro que documentamos aparte. Además,
    el filtro nuevo quitó una frase que era verdadera.
  - **No pudimos comprobar el caso con flujo neto negativo**, por un defecto en su preparación.
  - **Desviación de proceso, ya documentada:** el orden de entrega de los paquetes de la lectura
    principal y de K-FP(b) no quedó registrado. No cambia el veredicto.

  *Corrección del 05/11/2026:* la frase sobre la mempool reemplaza a la del commit `a6745f4` («Un texto
  daba por ausentes las comisiones de la mempool, que sí estaban en los datos»). El dato sí estaba en
  el corte, pero no llegaba al agente que escribió esa frase.

## Registro de cambios

Este repositorio **no reescribe su historial**: ni force-push ni commits borrados. Los errores se
corrigen con commits nuevos y se anotan aquí.

| Fecha (UTC) | Commit | Autor | Asunto (literal de git) | Cambios que no son rutina |
|---|---|---|---|---|
| 2026-11-04 03:42 | `e0453a5` | cryptolab-ledger-seals | README | A README.md |
| 2026-11-04 04:29 | `0a4cb14` | cryptolab-ledger-seals | README: cómo verificar y qué contiene el sello | README.md |
| 2026-11-04 04:33 | `b0ebfac` | cryptolab-ledger-seals | sellos 2026-11-04 04:33 UTC | A seals/2026-11-03.json.ots.bak |
| 2026-11-04 05:12 | `1443236` | cryptolab-ledger-seals | README: verificación en el navegador (opentimestamps.org) y script alternativo | README.md |
| 2026-11-04 05:13 | `06f269b` | cryptolab-ledger-seals | sellos 2026-11-04 05:13 UTC | D seals/2026-11-03.json.ots.bak |
| 2026-11-04 05:23 | `3760733` | cryptolab-ledger-seals | README: registro de cambios (06f269b, historial sin reescribir) y regla de anclaje anunciada | README.md |
| 2026-11-04 06:05 | `52082f4` | cryptolab-ledger-seals | README: sellos horarios, diario v2 y regla de anclaje completa | README.md |
| 2026-11-04 16:29 | `659ef24` | cryptolab-ledger | adenda P4-H: hash de ADENDA_P4_DESPLIEGUE_C0.md (cambio de versión del Técnico declarado antes) | A adendas/2026-11-04_p4h_despliegue_c0.json, A adendas/2026-11-04_p4h_despliegue_c0.json.ots |
| 2026-11-04 17:07 | `c770e72` | cryptolab-ledger-seals | sellos 2026-11-04 17:07 UTC | D adendas/2026-11-04_p4h_despliegue_c0.json, D adendas/2026-11-04_p4h_despliegue_c0.json.ots |
| 2026-11-04 18:41 | `1f865dc` | cryptolab-ledger | README: adendas (retiradas por c770e72, siguen en 659ef24), cadena canonica y preregistro P4-H2 | README.md |
| 2026-11-05 04:46 | `fc689dc` | cryptolab-ledger-seals | README: corrección de la frase de 1f865dc (la regla nunca tuvo corte por activación), ventana de exposición y hora tardía | README.md |
| 2026-11-05 18:28 | `a6745f4` | cryptolab-ledger-seals | README: registro de proceso (copia literal del apartado de la bitácora pública sobre la prueba de C0b v1.2, que falló); la bitácora completa se publicará tras la revisión legal | README.md |
| 2026-11-05 18:51 | `0fd7f8a` | cryptolab-ledger-seals | README: registro de cambios generado desde git log, notas sobre el historial y guía (el hash se comprueba sobre el raw, no sobre un checkout) | README.md |
| 2026-11-05 18:51 | `b6ee59f` | cryptolab-ledger-seals | README: registro de cambios regenerado desde git log tras 0fd7f8a | README.md |
| 2026-11-05 19:48 | `2bce0a1` | cryptolab-ledger-seals | README: corrección del apartado de proceso de C0b v1.2 (mempool: el dato estaba en el corte pero no llegaba al agente; reemplaza la frase de a6745f4), adenda 1 de P4-H2 y aclaración del hash de la bitácora (fichero interno aún no público) | README.md |

*Generado desde `git log` (UTC) hasta el commit `2bce0a1`. Los commits posteriores, incluido el que publica esta tabla, salen en la siguiente generación y en el historial de git.*

### Notas sobre el historial

Las filas de arriba salen de `git log`. Estas notas explican algunas; no sustituyen al historial.

- `c040301` (2026-11-04 03:43): primer sello (`2026-11-03`), con su `.ots` pendiente de Bitcoin. Es un commit de rutina y por eso no sale en la tabla.
- `b0ebfac`: `.ots` del 03/11 completado (bloques 974 473 y 974 476). **Error**: subió también `seals/2026-11-03.json.ots.bak`, la copia que `ots upgrade` deja del `.ots` pendiente anterior. No contenía datos, solo esa prueba antigua.
- `06f269b`: **retirado `seals/2026-11-03.json.ots.bak`.** Desde aquí la rutina solo publica `README.md`, `seals/*.json` y `seals/*.json.ots`. El `.bak` sigue visible en el historial de `b0ebfac`, **intencionadamente: no se reescribe el historial**.
- `6dee929` (2026-11-04 06:04): **primeros sellos horarios** (04/11, de 00:00 a 05:00, sellados todos en esta pasada). Es rutina.
- 2026-11-04 10:07: **subida fallida** durante la incidencia de GitHub en operaciones Git (09:49–11:30 UTC). El commit de los sellos de las 10:07 (`bf22811`) llegó con el de las 11:07 (`333164c`). No se perdió ni cambió ningún sello. La vigilancia P4-H lo contó como desincronía a las 10:20 (real) y a las 11:20 (error de medición: la consulta falló y no guardaba el mensaje de error).
- **Autor distinto en `659ef24` y `1f865dc`:** llevan `cryptolab-ledger <ledger@cryptolabdesk.invalid>`, y el resto de
  commits `cryptolab-ledger-seals <ledger-seals@users.noreply.github.com>`. Los dos se hicieron a mano el 04/11, fuera
  de la rutina automática, y quien los hizo configuró en el commit esa identidad de git en lugar de la de la rutina.
  No es otra cuenta: el push salió del mismo repositorio y con la misma clave de despliegue.
- `c770e72`: **error: la rutina de sellos retiró `adendas/`**, porque solo conserva `README.md` y `seals/`. La adenda sigue en `659ef24` (ver «Adendas y cadena canónica»). Se corrige con el cambio A0.
- `fc689dc`: **corrección**: la frase de `1f865dc` sobre las horas 00:00–05:00 del 04/11 era incorrecta (la regla nunca tuvo un corte por activación). Se añaden la ventana de exposición y la hora tardía, como presentación.
- `2bce0a1`: corrige el apartado de C0b v1.2 publicado en `a6745f4`. La bitácora pasa de `5d19e28a…` a `a6e135a1…`. Cambios de sustancia:
  - **«3 de los 7»**: antes, «Sí lo tuvieron 3 de los 7 textos de producción usados como comparación». Ahora se añade que **2 de esos 3 casos** se eligieron justamente por haber publicado ese error, así que no es una tasa.
  - **Quién hizo la lectura que dio el FAIL**: antes, «un lector independiente». Ahora, «un lector independiente (**otro modelo**)».
  - **Mempool**: antes, «daba por ausentes las comisiones de la mempool, que sí estaban en los datos». Ahora, estaban en el corte, pero ese agente no los recibía, por un defecto de construcción nuestro.
  - **Signo**: antes, «no tuvieron ningún error de signo». Ahora, «tras el filtro»: el filtro quitó uno que había escrito el modelo.
  - **Se añaden** dos cosas:
    - que la v1.1 también falló la revisión independiente y que nuestra primera revisión no lo detectó;
    - la desviación de proceso: no quedó registrado el orden de entrega de los paquetes, y eso no cambia el veredicto.
- **Las filas anteriores a este commit se escribieron a mano, con hora aproximada** (por ejemplo «06:1x», «18:4x»
  o «04:5x»). **Desde este commit, la tabla sale de `git log`**: se genera después de cada commit, en la misma
  pasada que la verificación por hash.
