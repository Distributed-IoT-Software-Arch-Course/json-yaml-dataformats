# SenML – Sensor Measurement Lists

This document introduces SenML, an IETF standard for representing **sensor measurements and device parameters**,
and shows how to create and parse SenML messages in Python using the standard `json` module.

[⬅ Previous: CSV](04-csv.md) · [Back to README](../README.md) · [Next: CBOR ➡](06-cbor.md)

- [What is SenML](#what-is-senml)
- [Why a standard for measurements](#why-a-standard-for-measurements)
- [SenML fields](#senml-fields)
- [Examples](#examples)
- [Resolving a SenML pack](#resolving-a-senml-pack)
- [Units](#units)
- [Python: create a SenML message](#python-create-a-senml-message)
- [Python: parse and resolve a SenML message](#python-parse-and-resolve-a-senml-message)
- [SenML representations: JSON, CBOR and others](#senml-representations-json-cbor-and-others)
- [Common pitfalls](#common-pitfalls)

## What is SenML

SenML (Sensor Measurement Lists) defines a simple data model and a set of representations for measurements
produced by sensors and for simple device parameters.

- Specification: [RFC 8428](https://www.rfc-editor.org/rfc/rfc8428)
- Media types: `application/senml+json`, `application/senml+cbor` (and others)
- Units registry: [IANA SenML Units](https://www.iana.org/assignments/senml/senml.xhtml)

A SenML message (called **pack**) is a JSON **array of records**. Each record is a JSON object with short field names.

```json
[
  {"n": "urn:dev:ow:10e2073a01080063", "u": "Cel", "v": 23.1}
]
```

The record above means: *the sensor `urn:dev:ow:10e2073a01080063` measured 23.1 degrees Celsius, now*.

## Why a standard for measurements

With plain JSON, different developers often define different structures for the same information:

```json
{"device_id": "temp-sensor-01", "value": 21.5, "unit": "C", "timestamp": 1727535600000}
```

```json
{"sensor": "temp-sensor-01", "temperature": 21.5, "ts": "2024-09-28T15:00:00Z"}
```

Both are valid JSON, but a consumer typically needs to know each specific structure (field names, unit symbols,
time format) to understand them. SenML standardizes these choices:

- standard field names (`n`, `v`, `u`, `t`, ...);
- standard **unit symbols** (`Cel`, `%RH`, `W`, `lx`, ...);
- standard **time** representation (seconds since the Unix epoch, as a number);
- **base values** to avoid repeating the same information in every record, helping keep messages small;
- the same model can be encoded in JSON, CBOR, XML and other formats.

SenML is used, for example, by the OMA **LwM2M** device management protocol, and is a common payload option for
**CoAP** and **MQTT** telemetry.

## SenML fields

**Base fields** (start with `b`) apply to the record where they appear **and to all the following records** in the pack,
until they are redefined:

| Field | Name | Type | Meaning |
|-------|------|------|---------|
| `bn`   | Base Name | String | Prefix added to the name of every record (e.g. the device identifier) |
| `bt`   | Base Time | Number | Time added to the time of every record |
| `bu`   | Base Unit | String | Unit used when a record does not specify `u` |
| `bv`   | Base Value | Number | Value added to the value of every record |
| `bs`   | Base Sum | Number | Value added to the sum of every record |
| `bver` | Base Version | Number | SenML version (default 10) |

**Regular fields** describe a single measurement:

| Field | Name | Type | Meaning |
|-------|------|------|---------|
| `n`  | Name | String | Name of the sensor/parameter (concatenated to `bn`) |
| `u`  | Unit | String | Unit of the value |
| `v`  | Value | Number | Numeric value |
| `vs` | String Value | String | Text value |
| `vb` | Boolean Value | Boolean | Boolean value |
| `vd` | Data Value | String | Binary data (base64url encoded) |
| `s`  | Sum | Number | Integrated sum of the values over time (e.g. total energy) |
| `t`  | Time | Number | Time of the measurement in seconds |
| `ut` | Update Time | Number | Maximum time before the sensor provides an updated value |

According to RFC 8428, a record has a (resolved) name and **exactly one** value field among `v`, `vs`, `vb`, `vd`
(or a sum `s`).

**Time** is a number in **seconds** (floats allowed for sub-second precision):
- values **≥ 2^28** (268,435,456) are **absolute** times: seconds since 1970-01-01 (Unix epoch);
- values **< 2^28** are **relative** to the current time: `0` (or missing) means *now*, `-5` means *5 seconds ago*.

## Examples

**Single measurement with absolute time**

```json
[
  {"n": "urn:dev:ow:10e2073a01080063", "u": "Cel", "v": 23.1, "t": 1727535600}
]
```

**Multiple sensors of the same device** (`bn` avoids repeating the device id; the name of each record becomes
`bn + n`):

```json
[
  {"bn": "urn:dev:mac:0024befffe804ff1:", "bt": 1727535600, "n": "temperature", "u": "Cel", "v": 21.5},
  {"n": "humidity", "u": "%RH", "v": 48.7},
  {"n": "battery", "u": "%EL", "v": 87},
  {"n": "door-open", "vb": false},
  {"n": "status", "vs": "OK"}
]
```

**Time series** of the same sensor (`bt`, `bu` and relative `t` keep each record very small):

```json
[
  {"bn": "urn:dev:mac:0024befffe804ff1:temperature", "bt": 1727535600, "bu": "Cel", "v": 21.5, "t": 0},
  {"v": 21.7, "t": 10},
  {"v": 21.4, "t": 20},
  {"v": 21.3, "t": 30}
]
```

**Energy meter** with both the instantaneous power and the cumulative energy (sum):

```json
[
  {"bn": "urn:dev:mac:0024befffe804ff2:", "n": "power", "u": "W", "v": 1250.5},
  {"n": "energy", "u": "J", "s": 45810000}
]
```

## Resolving a SenML pack

**Resolving** a pack means applying the base fields to every record, obtaining self-contained records:

- name = `bn` + `n`
- time = `bt` + `t` (and, if the result is relative, converted to absolute)
- unit = `u` if present, otherwise `bu`
- value = `bv` + `v`

The time series example above resolves to:

```json
[
  {"n": "urn:dev:mac:0024befffe804ff1:temperature", "u": "Cel", "v": 21.5, "t": 1727535600},
  {"n": "urn:dev:mac:0024befffe804ff1:temperature", "u": "Cel", "v": 21.7, "t": 1727535610},
  {"n": "urn:dev:mac:0024befffe804ff1:temperature", "u": "Cel", "v": 21.4, "t": 1727535620},
  {"n": "urn:dev:mac:0024befffe804ff1:temperature", "u": "Cel", "v": 21.3, "t": 1727535630}
]
```

The compact form is used on the network, the resolved form is easier to process and store (e.g. in a database).

## Units

SenML units come from an IANA registry and are mostly SI symbols. Some common ones:

| Unit | Meaning | Unit | Meaning |
|------|---------|------|---------|
| `Cel` | degrees Celsius | `%RH` | relative humidity (%) |
| `K`   | kelvin | `%EL` | remaining battery energy level (%) |
| `Pa`  | pascal (pressure) | `lx` | lux (illuminance) |
| `W`   | watt (power) | `J`  | joule (energy) |
| `V`   | volt | `A`  | ampere |
| `m`   | meter | `m/s` | meters per second |
| `s`   | second | `/` | ratio (e.g. 0.5) |
| `lat` | degrees latitude | `lon` | degrees longitude |
| `Hz`  | hertz (frequency) | `%` | percent |

Note that Celsius is `Cel` (not `C`, which is the coulomb!).

## Python: create a SenML message

No special library is needed: a SenML pack is a list of dictionaries serialized with `json`.

```python
import json
import time

def build_senml_pack(device_id, measurements):
    """measurements: list of (name, value, unit) tuples measured at the same instant."""
    pack = []
    for i, (name, value, unit) in enumerate(measurements):
        record = {"n": name}
        if unit is not None:
            record["u"] = unit
        # choose the value field according to the Python type (bool must be checked before int!)
        if isinstance(value, bool):
            record["vb"] = value
        elif isinstance(value, (int, float)):
            record["v"] = value
        else:
            record["vs"] = str(value)
        if i == 0:   # base fields only in the first record
            record = {"bn": f"{device_id}:", "bt": int(time.time()), **record}
        pack.append(record)
    return pack

pack = build_senml_pack("urn:dev:mac:0024befffe804ff1", [
    ("temperature", 21.5, "Cel"),
    ("humidity", 48.7, "%RH"),
    ("door-open", False, None),
])

payload = json.dumps(pack)
print(payload)
```

Example output:

```json
[{"bn": "urn:dev:mac:0024befffe804ff1:", "bt": 1727535600, "n": "temperature", "u": "Cel", "v": 21.5}, {"n": "humidity", "u": "%RH", "v": 48.7}, {"n": "door-open", "vb": false}]
```

The payload can now be published, for example, on an MQTT topic or returned by a CoAP resource with
content-format `application/senml+json` (CoAP content-format number `110`).

## Python: parse and resolve a SenML message

The following function implements the resolution rules described above:

```python
import json
import time

SENML_RELATIVE_TIME_LIMIT = 2 ** 28   # times below this value are relative to "now"
VALUE_FIELDS = ("v", "vs", "vb", "vd", "s")

def resolve_senml(pack, now=None):
    """Applies base fields to every record and returns a list of self-contained records."""
    now = time.time() if now is None else now
    base_name, base_time, base_unit, base_value = "", 0, None, 0
    resolved = []
    for record in pack:
        # update the base values (they also apply to the following records)
        base_name = record.get("bn", base_name)
        base_time = record.get("bt", base_time)
        base_unit = record.get("bu", base_unit)
        base_value = record.get("bv", base_value)

        r = {"n": base_name + record.get("n", "")}
        unit = record.get("u", base_unit)
        if unit is not None:
            r["u"] = unit
        if "v" in record:
            r["v"] = base_value + record["v"]
        for field in ("vs", "vb", "vd", "s"):
            if field in record:
                r[field] = record[field]
        t = base_time + record.get("t", 0)
        r["t"] = t + now if t < SENML_RELATIVE_TIME_LIMIT else t

        if not any(field in r for field in VALUE_FIELDS):
            raise ValueError(f"SenML record without value: {record}")
        resolved.append(r)
    return resolved

payload = '''[
  {"bn": "urn:dev:mac:0024befffe804ff1:", "bt": 1727535600, "n": "temperature", "u": "Cel", "v": 21.5},
  {"n": "humidity", "u": "%RH", "v": 48.7, "t": 2},
  {"n": "door-open", "vb": false, "t": 5}
]'''

for record in resolve_senml(json.loads(payload)):
    print(record)
```

Output:

```text
{'n': 'urn:dev:mac:0024befffe804ff1:temperature', 'u': 'Cel', 'v': 21.5, 't': 1727535600}
{'n': 'urn:dev:mac:0024befffe804ff1:humidity', 'u': '%RH', 'v': 48.7, 't': 1727535602}
{'n': 'urn:dev:mac:0024befffe804ff1:door-open', 'vb': False, 't': 1727535605}
```

Once resolved, the records are easy to store or convert, for example into rows of a [CSV](04-csv.md) file.

> The helper above covers the most common cases (`bn`, `bt`, `bu`, `bv`). A complete implementation should also handle
> `bs`, `bver` and name validation as described in RFC 8428.

## SenML representations: JSON, CBOR and others

The SenML data model can be encoded in different formats. For constrained devices the **CBOR** representation
(`application/senml+cbor`, CoAP content-format `112`) is usually more compact, because field names are replaced by small
integers:

| JSON label | `bver` | `bn` | `bt` | `bu` | `bv` | `bs` | `n` | `u` | `v` | `vs` | `vb` | `s` | `t` | `ut` | `vd` |
|------------|--------|------|------|------|------|------|-----|-----|-----|------|------|-----|-----|------|------|
| CBOR label | -1 | -2 | -3 | -4 | -5 | -6 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |

See the [CBOR document](06-cbor.md#example-senml-in-cbor) for a Python example. RFC 8428 also defines XML and EXI
representations, which are less commonly used in practice.

## Common pitfalls

- A SenML pack is an **array**, even when it contains a single record.
- Time is in **seconds**, not milliseconds (a common mistake when coming from JavaScript `Date.now()`).
- Registered units are expected: `Cel` rather than `C` or `°C`; `%RH` for humidity.
- Base fields are "sticky": they apply to all the following records until redefined.
- A record is expected to contain exactly one value field (`v`, `vs`, `vb`, `vd`) or a sum `s`.
- Using the specific media type `application/senml+json` helps receivers know how to interpret the payload.

---

[⬅ Previous: CSV](04-csv.md) · [Back to README](../README.md) · [Next: CBOR ➡](06-cbor.md)
