# Formats Comparison – Which Format for Which Job

This document puts the formats side by side, representing the **same data** in each of them, and provides a quick
guide to help choose among them.

[⬅ Previous: Data Validation](07-validation.md) · [Back to README](../README.md)

- [The same configuration in different formats](#the-same-configuration-in-different-formats)
- [The same measurements in different formats](#the-same-measurements-in-different-formats)
- [Feature comparison](#feature-comparison)
- [Decision guide](#decision-guide)
- [Python cheat sheet](#python-cheat-sheet)

## The same configuration in different formats

A sensor configuration: device id, broker settings and a list of units.

**JSON**

```json
{
  "device_id": "temp-sensor-01",
  "enabled": true,
  "mqtt": {"host": "localhost", "port": 1883},
  "units": ["Cel", "K"]
}
```

**YAML**

```yaml
# Kitchen temperature sensor
device_id: temp-sensor-01
enabled: true
mqtt:
  host: localhost
  port: 1883
units: [Cel, K]
```

**TOML**

```toml
# Kitchen temperature sensor
device_id = "temp-sensor-01"
enabled = true
units = ["Cel", "K"]

[mqtt]
host = "localhost"
port = 1883
```

**CBOR** (binary, shown as hex – 77 bytes vs 105 of the compact JSON)

```text
a4 69 64 65 76 69 63 65 5f 69 64 6e 74 65 6d 70 2d 73 65 6e 73 6f 72 2d 30 31 67 65 6e 61 62 6c
65 64 f5 64 6d 71 74 74 a2 64 68 6f 73 74 69 6c 6f 63 61 6c 68 6f 73 74 64 70 6f 72 74 19 07 5b
65 75 6e 69 74 73 82 63 43 65 6c 61 4b
```

CSV and SenML are generally not well suited for this kind of nested configuration data.

## The same measurements in different formats

Two sensors of the same device, measured at the same time.

**JSON** (custom structure)

```json
{
  "device_id": "env-node-01",
  "timestamp": 1727535600,
  "measurements": [
    {"name": "temperature", "value": 21.5, "unit": "Cel"},
    {"name": "humidity", "value": 48.7, "unit": "%RH"}
  ]
}
```

**YAML** (possible, but less common for messages)

```yaml
device_id: env-node-01
timestamp: 1727535600
measurements:
  - {name: temperature, value: 21.5, unit: Cel}
  - {name: humidity, value: 48.7, unit: "%RH"}
```

**CSV** (one row per measurement, often used for logs)

```csv
device_id,timestamp,name,value,unit
env-node-01,1727535600,temperature,21.5,Cel
env-node-01,1727535600,humidity,48.7,%RH
```

**SenML-JSON** (standard structure)

```json
[
  {"bn": "env-node-01:", "bt": 1727535600, "n": "temperature", "u": "Cel", "v": 21.5},
  {"n": "humidity", "u": "%RH", "v": 48.7}
]
```

**SenML-CBOR** (binary, integer labels: `-2` = `bn`, `-3` = `bt`, `0` = `n`, `1` = `u`, `2` = `v`)

```text
[{-2: "env-node-01:", -3: 1727535600, 0: "temperature", 1: "Cel", 2: 21.5}, {0: "humidity", 1: "%RH", 2: 48.7}]
```

## Feature comparison

The table summarizes typical characteristics; actual results may vary depending on the data and the libraries used.

| | JSON | YAML | TOML | CSV | SenML | CBOR |
|---|---|---|---|---|---|---|
| Encoding | text | text | text | text | JSON / CBOR / XML | **binary** |
| Human readable | ✅ | ✅✅ | ✅✅ | ✅ | ✅ (JSON) | ❌ |
| Comments | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Nesting | ✅ | ✅ | ✅ (can be verbose) | ❌ | flat records | ✅ |
| Explicit types | basic | implicit | ✅ (+ dates) | ❌ (all strings) | fixed fields | rich (bytes, tags) |
| Size | medium | medium | medium | usually small (tables) | usually small | usually smallest |
| Schema options | JSON Schema | JSON Schema | JSON Schema | – | defined by RFC | CDDL / JSON Schema |
| Python support | `json` (built-in) | `PyYAML` | `tomllib` (built-in, read) | `csv` (built-in) | `json` / `cbor2` | `cbor2` |
| Common IoT uses | messages, APIs, telemetry | configuration, deployment | app settings | logs, datasets | sensor measurements | constrained devices, CoAP |

## Decision guide

A simplified guide based on common practice. Real projects can have good reasons to make different choices
(existing tools, team habits, interoperability requirements).

```text
Who usually writes / reads the data?
│
├─ Humans (configuration)
│     ├─ deeply nested, or expected by the tool (Docker, K8s, CI) ──▶ often YAML
│     └─ mostly key/value app settings, Python project ─────────────▶ often TOML
│
└─ Machines (data exchange)
      ├─ messages, events, commands, REST APIs (common default) ────▶ often JSON
      ├─ tabular data, logs, datasets for analysis ─────────────────▶ often CSV
      ├─ sensor measurements to be shared across vendors ───────────▶ consider SenML
      │        └─ on constrained devices / CoAP ────────────────────▶ consider SenML-CBOR
      └─ constrained device or network, binary data ────────────────▶ consider CBOR
```

Whatever the format, it is generally good practice to define a **schema**, **validate** data at the boundaries
([Data Validation](07-validation.md)) and **document** conventions (units, time format, field names).

## Python cheat sheet

| Format | Install | Read | Write | File mode |
|--------|---------|------|-------|-----------|
| JSON | built-in | `json.load(f)` / `json.loads(s)` | `json.dump(obj, f)` / `json.dumps(obj)` | text |
| YAML | `pip install pyyaml` | `yaml.safe_load(f)` | `yaml.safe_dump(obj, f, sort_keys=False)` | text |
| TOML | built-in (read) / `pip install tomli-w` | `tomllib.load(f)` | `tomli_w.dump(obj, f)` | **binary** |
| CSV | built-in | `csv.DictReader(f)` | `csv.DictWriter(f, fieldnames)` | text, `newline=""` |
| SenML | – | `json.loads(s)` + resolve records | `json.dumps(pack)` | text |
| CBOR | `pip install cbor2` | `cbor2.load(f)` / `cbor2.loads(b)` | `cbor2.dump(obj, f)` / `cbor2.dumps(obj)` | **binary** |

---

[⬅ Previous: Data Validation](07-validation.md) · [Back to README](../README.md)
