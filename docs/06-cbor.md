# CBOR – Concise Binary Object Representation

This document introduces CBOR, a **binary** data format based on the JSON data model, designed for constrained devices
and networks, and shows how to use it in Python with the `cbor2` library.

[⬅ Previous: SenML](05-senml.md) · [Back to README](../README.md) · [Next: Data Validation ➡](07-validation.md)

- [What is CBOR](#what-is-cbor)
- [Why a binary format](#why-a-binary-format)
- [Data model and types](#data-model-and-types)
- [Where CBOR is used in IoT](#where-cbor-is-used-in-iot)
- [Python: the cbor2 library](#python-the-cbor2-library)
- [Encode and decode](#encode-and-decode)
- [Read and write CBOR files](#read-and-write-cbor-files)
- [Size comparison with JSON](#size-comparison-with-json)
- [Looking inside a CBOR message](#looking-inside-a-cbor-message)
- [Example: SenML in CBOR](#example-senml-in-cbor)
- [Common pitfalls](#common-pitfalls)

## What is CBOR

CBOR is a binary serialization format whose data model is a **superset of JSON**: everything that can be represented in
JSON can be represented in CBOR, plus binary data, more number types and extensible tags (e.g., dates).

- Specification: [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949) and [cbor.io](https://cbor.io/)
- Media type: `application/cbor` (CoAP content-format `60`)
- File extension: `.cbor`

You can think of CBOR as a kind of "**binary JSON**": the structure of the data is largely the same (maps, arrays,
strings, numbers, booleans, null), while the encoding on the wire changes.

## Why a binary format

JSON is text: the number `1727535600` takes 10 bytes (one per digit), every key is repeated as a quoted string, and the
receiver needs to parse characters to rebuild numbers. On a battery-powered sensor transmitting over a low-power radio
network (e.g., LoRaWAN, NB-IoT, 6LoWPAN), bytes and CPU cycles can matter a lot.

Some CBOR advantages:

- **Smaller messages** (in many cases): numbers are stored in binary using few bytes, small integers fit in one byte.
- **Typically faster and simpler parsing**: each item starts with a header byte describing its type and length, so
  there is less need to scan characters, escape strings or convert text to numbers. Encoders/decoders can fit in a
  few KB of firmware.
- **Native binary data** (byte strings) without base64 encoding (which adds about 33% overhead in JSON).
- **Schema-less** like JSON: there is no need to define and compile a schema (unlike, e.g., Protocol Buffers).

The main drawback: it is **not human-readable**, so a tool is usually needed to inspect it (e.g., [cbor.me](https://cbor.me/)).

## Data model and types

| CBOR major type | Description | JSON equivalent | Python type (`cbor2`) |
|-----------------|-------------|-----------------|------------------------|
| 0 / 1 | Unsigned / negative integer | number | `int` |
| 2 | Byte string | – (base64 string) | `bytes` |
| 3 | Text string (UTF-8) | string | `str` |
| 4 | Array | array | `list` |
| 5 | Map | object | `dict` |
| 6 | Tag (semantic meaning, e.g. date) | – | e.g. `datetime` |
| 7 | Float, `true`, `false`, `null` | number, boolean, null | `float`, `bool`, `None` |

Unlike JSON, CBOR map keys can be **any type**, including integers. This is used by many IoT standards
(SenML-CBOR, COSE, CWT) to replace long string keys with small integers.

## Where CBOR is used in IoT

Some examples:

- **CoAP** payloads on constrained devices (`application/cbor`, `application/senml+cbor`).
- **SenML-CBOR** sensor measurements ([see below](#example-senml-in-cbor)).
- **LwM2M** device management (SenML-CBOR content format).
- **Security**: COSE (CBOR Object Signing and Encryption) and CWT (CBOR Web Token, the compact version of JWT).
- **WebAuthn / FIDO2** (passkeys) uses CBOR for authenticator data.

## Python: the cbor2 library

```bash
pip install "cbor2>=5.6,<6"
```

> The 6.x versions of `cbor2` are built in Rust: if no pre-built package is available for your platform, the
> installation requires a Rust compiler. The 5.x series works everywhere and has the same API used here.

The API mirrors the `json` module, but works with **bytes** instead of strings:

| Function | From | To |
|----------|------|----|
| `cbor2.load(f)` | CBOR file (opened in `rb`) | Python object |
| `cbor2.dump(obj, f)` | Python object | CBOR file (opened in `wb`) |
| `cbor2.loads(data)` | `bytes` | Python object |
| `cbor2.dumps(obj)` | Python object | `bytes` |

## Encode and decode

```python
import cbor2

measurement = {"device_id": "temp-sensor-01", "value": 21.5, "unit": "C",
               "timestamp": 1727535600000, "online": True}

payload = cbor2.dumps(measurement)        # dict -> bytes (ready to be sent with CoAP/MQTT)
print(type(payload), len(payload))        # Output: <class 'bytes'> 75

decoded = cbor2.loads(payload)            # bytes -> dict
print(decoded == measurement)             # Output: True
print(decoded["value"])                   # Output: 21.5
```

CBOR natively supports binary data and dates:

```python
import cbor2
from datetime import datetime, timezone

firmware_chunk = {
    "device_id": "temp-sensor-01",
    "chunk": 3,
    "data": bytes([0xDE, 0xAD, 0xBE, 0xEF]),                      # raw bytes, no base64 needed
    "created": datetime(2024, 9, 28, 15, 0, tzinfo=timezone.utc),  # encoded with a CBOR date tag
}

decoded = cbor2.loads(cbor2.dumps(firmware_chunk))
print(decoded["data"])      # Output: b'\xde\xad\xbe\xef'
print(decoded["created"])   # Output: 2024-09-28 15:00:00+00:00
```

## Read and write CBOR files

CBOR is binary, so files are opened in **binary mode** (`wb` / `rb`):

```python
import cbor2

measurements = [
    {"device_id": "temp-sensor-01", "value": 21.5, "timestamp": 1727535600},
    {"device_id": "temp-sensor-01", "value": 21.7, "timestamp": 1727535601},
]

with open("measurements.cbor", "wb") as f:
    cbor2.dump(measurements, f)

with open("measurements.cbor", "rb") as f:
    loaded = cbor2.load(f)

print(loaded[1]["value"])   # Output: 21.7
```

## Size comparison with JSON

```python
import json
import cbor2

measurement = {"device_id": "temp-sensor-01", "value": 21.5, "unit": "C",
               "timestamp": 1727535600000, "online": True}

json_bytes = json.dumps(measurement, separators=(",", ":")).encode("utf-8")   # compact JSON
cbor_bytes = cbor2.dumps(measurement)
cbor_canonical = cbor2.dumps(measurement, canonical=True)   # smallest float size + sorted keys

print("JSON :", len(json_bytes), "bytes")       # Output: JSON : 94 bytes
print("CBOR :", len(cbor_bytes), "bytes")       # Output: CBOR : 75 bytes
print("CBOR (canonical):", len(cbor_canonical), "bytes")   # Output: CBOR (canonical): 69 bytes
```

The saving here is about 20-25%, because a large part of the message is made of **text** (keys and the device id),
which takes the same space in both formats. The gain usually becomes larger when:

- messages contain mostly **numbers** or **binary data**;
- string keys are replaced by **integer keys** (as SenML-CBOR does);
- the same structure is repeated many times (time series).

> CBOR can be combined with compression, but on very small messages (tens of bytes) compression is usually not very effective,
> while CBOR can still reduce the size.

## Looking inside a CBOR message

Let's encode a tiny message and print the bytes in hexadecimal:

```python
import cbor2

print(cbor2.dumps({"v": 21.5, "t": 1727535600}).hex(" "))
# Output: a2 61 76 fb 40 35 80 00 00 00 00 00 61 74 1a 66 f8 19 f0
```

| Bytes | Meaning |
|-------|---------|
| `a2` | map (major type 5) with 2 pairs |
| `61 76` | text string of length 1: `"v"` |
| `fb 40 35 80 00 00 00 00 00` | 64-bit float: `21.5` |
| `61 74` | text string of length 1: `"t"` |
| `1a 66 f8 19 f0` | 32-bit unsigned integer: `1727535600` |

19 bytes against the 25 of the compact JSON `{"v":21.5,"t":1727535600}`. With `canonical=True` the float `21.5` is
encoded as a 16-bit half-precision float (`f9 4d 60`, 3 bytes instead of 9) and the message becomes 14 bytes.
You can paste the hex string on [cbor.me](https://cbor.me/) to decode it interactively.

## Example: SenML in CBOR

SenML-CBOR (RFC 8428, Section 6) uses the same records as [SenML-JSON](05-senml.md), but replaces the field names
with integer labels (`bn` → `-2`, `bt` → `-3`, `n` → `0`, `u` → `1`, `v` → `2`, ...):

```python
import json
import cbor2

SENML_CBOR_LABELS = {"bver": -1, "bn": -2, "bt": -3, "bu": -4, "bv": -5, "bs": -6,
                     "n": 0, "u": 1, "v": 2, "vs": 3, "vb": 4, "s": 5, "t": 6, "ut": 7, "vd": 8}
SENML_JSON_LABELS = {label: name for name, label in SENML_CBOR_LABELS.items()}

def senml_to_cbor(pack):
    return cbor2.dumps([{SENML_CBOR_LABELS[k]: v for k, v in record.items()} for record in pack])

def senml_from_cbor(data):
    return [{SENML_JSON_LABELS[k]: v for k, v in record.items()} for record in cbor2.loads(data)]

pack = [
    {"bn": "urn:dev:mac:0024befffe804ff1:", "bt": 1727535600, "n": "temperature", "u": "Cel", "v": 21.5},
    {"n": "humidity", "u": "%RH", "v": 48.7},
]

senml_json = json.dumps(pack, separators=(",", ":")).encode("utf-8")
senml_cbor = senml_to_cbor(pack)

print("SenML-JSON:", len(senml_json), "bytes")   # Output: SenML-JSON: 129 bytes
print("SenML-CBOR:", len(senml_cbor), "bytes")   # Output: SenML-CBOR: 94 bytes
print(senml_from_cbor(senml_cbor) == pack)       # Output: True
```

> Note: SenML-CBOR (with `Content-Format: application/senml+cbor`, `112`) is a different media type than generic CBOR
> (`application/cbor`, `60`): the receiver needs to know that integer keys are SenML labels.

## Common pitfalls

- Files are opened in **binary** mode (`rb`/`wb`); CBOR data is `bytes`, not `str`.
- CBOR is not human-readable: logging the decoded object (or the hex dump) usually helps when debugging.
- Not every CBOR value can be converted to JSON (byte strings, integer keys, tags): conversions need some care.
- Both sides need to agree on conventions (e.g., integer key mapping, date representation), just like with JSON.
- For tiny devices it is worth checking which features are supported by the embedded library (e.g., floats, tags, indefinite lengths).

---

[⬅ Previous: SenML](05-senml.md) · [Back to README](../README.md) · [Next: Data Validation ➡](07-validation.md)
