# TOML – Tom's Obvious, Minimal Language

This document introduces TOML, a configuration format designed to be simple and unambiguous, and shows how to read it
with Python's built-in `tomllib` module (and write it with `tomli-w`).

- [What is TOML](#what-is-toml)
- [Syntax](#syntax)
- [Where TOML is used](#where-toml-is-used)
- [Python: read TOML with tomllib](#python-read-toml-with-tomllib)
- [Python: write TOML with tomli-w](#python-write-toml-with-tomli-w)
- [TOML vs YAML for configuration](#toml-vs-yaml-for-configuration)
- [Common pitfalls](#common-pitfalls)

## What is TOML

TOML is a configuration file format designed to map unambiguously to a dictionary (hash table). It looks similar to
the classic `.ini` files, but with a precise specification and **explicit types**.

- Specification: [toml.io](https://toml.io/en/v1.0.0) (version 1.0.0)
- Media type: `application/toml`
- File extension: `.toml`

```toml
# Gateway configuration
gateway_id = "gw-01"
log_level = "INFO"

[mqtt]
host = "localhost"
port = 1883
tls = false
```

## Syntax

**Key/value pairs**: `key = value`. Strings are quoted, which helps avoid ambiguity about types:

```toml
name = "temp-sensor-01"     # string
port = 1883                 # integer
threshold = 25.5            # float
enabled = true              # boolean
version = "1.10"            # string (quoted, so it stays a string)
installed = 2024-09-28T15:00:00Z   # date-time (native type!)
units = ["Cel", "K"]        # array
position = { lat = 44.6471, lon = 10.9252 }   # inline table
```

TOML has **no null** value: a missing key is usually interpreted as "not set".

**Tables** (sections) group keys, like nested dictionaries. Dotted names create nested tables:

```toml
[mqtt]
host = "localhost"
port = 1883

[mqtt.auth]          # nested table: config["mqtt"]["auth"]
username = "gateway"
password_env = "MQTT_PASSWORD"
```

**Arrays of tables** (`[[...]]`) define lists of objects, e.g. a list of devices:

```toml
[[devices]]
device_id = "temp-sensor-01"
type = "TemperatureSensor"
sampling_period_ms = 1000

[[devices]]
device_id = "hum-sensor-02"
type = "HumiditySensor"
sampling_period_ms = 5000
```

## Where TOML is used

Some common examples:

- **Python projects**: `pyproject.toml` (project metadata, dependencies, tool settings for pytest, ruff, black...).
- **Rust projects**: `Cargo.toml`.
- **Application configuration**: Telegraf (the InfluxData agent, often used for IoT metrics collection),
  InfluxDB, Hugo, Gitea, Poetry, uv...

Example: a fragment of a Telegraf configuration that subscribes to MQTT telemetry:

```toml
[[inputs.mqtt_consumer]]
servers = ["tcp://localhost:1883"]
topics = ["home/sensors/#"]
data_format = "json"
```

## Python: read TOML with tomllib

Since Python 3.11, the standard library includes `tomllib` (read-only). Note that the file must be opened in
**binary mode** (`"rb"`).

Given a `gateway_config.toml` file:

```toml
gateway_id = "gw-01"
installed = 2024-09-28T15:00:00Z

[mqtt]
host = "localhost"
port = 1883
base_topic = "home/sensors"

[[devices]]
device_id = "temp-sensor-01"
type = "TemperatureSensor"
sampling_period_ms = 1000

[[devices]]
device_id = "hum-sensor-02"
type = "HumiditySensor"
sampling_period_ms = 5000
```

```python
import tomllib

with open("gateway_config.toml", "rb") as f:
    config = tomllib.load(f)

print(config["mqtt"]["host"], config["mqtt"]["port"])   # Output: localhost 1883
print(repr(config["installed"]))
# Output: datetime.datetime(2024, 9, 28, 15, 0, tzinfo=datetime.timezone.utc)

for device in config["devices"]:
    print(device["device_id"], device["sampling_period_ms"])
```

Output:

```text
localhost 1883
datetime.datetime(2024, 9, 28, 15, 0, tzinfo=datetime.timezone.utc)
temp-sensor-01 1000
hum-sensor-02 5000
```

From a string, use `tomllib.loads()`:

```python
import tomllib

config = tomllib.loads('log_level = "DEBUG"\n[mqtt]\nport = 8883')
print(config)   # Output: {'log_level': 'DEBUG', 'mqtt': {'port': 8883}}
```

> For Python < 3.11 install the `tomli` package: it has the same API (`import tomli as tomllib`).

## Python: write TOML with tomli-w

`tomllib` can only read. To write TOML files install `tomli-w`:

```bash
pip install tomli-w
```

```python
import tomli_w

config = {
    "gateway_id": "gw-02",
    "mqtt": {"host": "broker.example.com", "port": 8883, "tls": True},
    "devices": [
        {"device_id": "temp-sensor-05", "type": "TemperatureSensor", "sampling_period_ms": 2000},
    ],
}

with open("new_gateway_config.toml", "wb") as f:   # binary mode here too
    tomli_w.dump(config, f)

print(tomli_w.dumps(config))
```

Output:

```toml
gateway_id = "gw-02"
devices = [
    { device_id = "temp-sensor-05", type = "TemperatureSensor", sampling_period_ms = 2000 },
]

[mqtt]
host = "broker.example.com"
port = 8883
tls = true
```

`tomli-w` writes the list of devices as an array of **inline tables**: it is equivalent to the `[[devices]]` syntax
shown above (`tomllib` loads both into the same Python list of dictionaries).

## TOML vs YAML for configuration

| | TOML | YAML |
|---|------|------|
| Readability | Usually very good for flat or 2-level configs | Usually very good, also for deep nesting |
| Typing | Explicit (strings are quoted) | Implicit (possible surprises like `NO` → `false`) |
| Deep nesting | Can become verbose (`[a.b.c]`) | Natural (indentation) |
| Comments | ✅ | ✅ |
| Dates | Native type | Supported (implicit) |
| Null | ❌ | ✅ |
| Python support | Read: built-in (`tomllib`); write: `tomli-w` | `PyYAML` (external) |
| Common uses | `pyproject.toml`, app settings | docker-compose, Kubernetes, CI pipelines |

As a general rule of thumb, TOML is often a good fit for **application settings** that are mostly key/value, while
YAML is usually preferred for **deeply nested** configurations or when the ecosystem already expects it
(Docker, Kubernetes, GitHub Actions...). The choice also depends on team habits and tooling.

## Common pitfalls

- `tomllib` requires files opened in **binary mode** (`"rb"`); `tomli_w.dump` requires `"wb"`.
- `tomllib` is read-only: a separate library such as `tomli-w` is needed to write. Comments are typically lost when writing (like YAML with PyYAML).
- There is no `null`: `tomli_w` cannot serialize `None` values, so those keys are usually removed before writing.
- A table can be defined only once: repeating `[mqtt]` twice in the same file is an error.
- Keys after a `[table]` header belong to that table until the next header: it is common practice to put top-level keys **before** any table.
