# cryptolab-ledger-seals

Sellos diarios encadenados del registro de lecturas de CryptoLab, anclados en Bitcoin con
[OpenTimestamps](https://opentimestamps.org).

**Este repositorio solo contiene hashes, recuentos, fechas y las pruebas `.ots`.** No contiene
lecturas, textos, precios ni datos de ningún proveedor. Sirve para que cualquiera pueda comprobar
dos cosas:

1. Que el registro de un día **existía tal cual en una fecha**, porque el `.ots` ancla el hash en
   un bloque de Bitcoin.
2. Que **ningún día anterior se ha reescrito**, porque cada sello incluye el hash del anterior.

## Contenido

| Fichero | Qué es |
|---|---|
| `seals/AAAA-MM-DD.json` | Sello del día UTC `AAAA-MM-DD` |
| `seals/AAAA-MM-DD.json.ots` | Prueba OpenTimestamps del fichero anterior |

Campos de cada sello:

| Campo | Contenido |
|---|---|
| `seal_date` | día UTC sellado (lo capturado ese día) |
| `sealed_at` | cuándo se calculó el sello (siempre después de terminar el día) |
| `judge_rows` | número de veredictos del Judge capturados ese día |
| `judge_digest` | hash de las filas del Judge del día, que **incluye el texto completo de cada brief** (ver abajo) |
| `layer_rows` | número de lecturas capturadas por capa (técnico, on-chain, macro) |
| `layer_digest` | hash de las lecturas de capa del día |
| `prev_seal_hash` | `seal_hash` del sello anterior (`null` en el primero) |
| `seal_hash` | hash del propio sello |
| `algoritmo` | referencia a esta descripción |

## Algoritmo

`H(p1, …, pn)` = SHA-256 de la concatenación de cada parte en UTF-8, seguida del byte `0x1f`. Los
objetos JSON se serializan con claves ordenadas, sin espacios y con `ensure_ascii=false`.

| Hash | Cálculo |
|---|---|
| `judge_digest` | `H(f₁, …, f_k)` con `f_i = "<row_hash_i>:<H(brief_i)>"`, para las filas del Judge capturadas ese día, ordenadas por hora de captura. `row_hash_i` encadena cada fila con la anterior en el registro; `H(brief_i)` cubre el texto íntegro del brief |
| `layer_digest` | igual, con `"<row_hash_i>:<payload_sha256_i>"` de las lecturas de capa |
| `seal_hash` | `H(seal_date, judge_digest, layer_digest, prev_seal_hash or "")` |

Las filas de origen están en un registro **append-only**: la base de datos rechaza UPDATE y DELETE.
Pueden mostrarse a un auditor para que recalcule estos hashes.

## Cómo verificar

```bash
pip install opentimestamps-client
ots verify seals/AAAA-MM-DD.json.ots     # requiere un nodo de Bitcoin o usa: ots --no-bitcoin info …
```

Para comprobar la cadena, `prev_seal_hash` de cada día debe ser igual al `seal_hash` del día
anterior.

Las pruebas `.ots` de los últimos días pueden estar **pendientes**: el anclaje en Bitcoin tarda
unas horas. Se completan solas en la ejecución de la noche siguiente.
