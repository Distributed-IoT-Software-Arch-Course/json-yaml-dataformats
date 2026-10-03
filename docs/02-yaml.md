# YAML – YAML Ain't Markup Language

This document introduces YAML, one of the formats most commonly used for **configuration files**, and shows how to read and write it
in Python with the PyYAML library.

[⬅ Previous: JSON](01-json.md) · [Back to README](../README.md) · [Next: TOML ➡](03-toml.md)

- [What is YAML](#what-is-yaml)
- [Basic syntax](#basic-syntax)
- [Advanced features](#advanced-features)
- [Where YAML is used in IoT](#where-yaml-is-used-in-iot)
- [Python: the PyYAML library](#python-the-pyyaml-library)
- [Load a YAML configuration file](#load-a-yaml-configuration-file)
- [Write a YAML file](#write-a-yaml-file)
- [Multiple documents in one file](#multiple-documents-in-one-file)
- [Converting between YAML and JSON](#converting-between-yaml-and-json)
- [Common pitfalls](#common-pitfalls)

## What is YAML

YAML is a human-friendly data serialization format. It represents the same data model as JSON (mappings, sequences and
scalar values) but uses **indentation** instead of braces and brackets, and supports **comments**.
These features make it a frequent choice for files written and maintained **by humans**, such as configurations.

- Specification: [yaml.org](https://yaml.org/) (current version 1.2)
- Media type: `application/yaml`
- File extensions: `.yaml` or `.yml`

The same data in JSON and YAML:

```json
{
  "device_id": "temp-sensor-01",
  "sampling_period_ms": 1000,
  "units": ["C", "F"],
  "mqtt": {"host": "localhost", "port": 1883}
}
```

```yaml
# Temperature sensor configuration
device_id: temp-sensor-01
sampling_period_ms: 1000
units:
  - C
  - F
mqtt:
  host: localhost
  port: 1883
```

> YAML 1.2 is (almost) a **superset of JSON**: a valid JSON document is also a valid YAML document.

## Basic syntax

**Mappings** (key-value pairs, like JSON objects / Python dictionaries) use `key: value`.
Nesting is expressed with indentation (**spaces only, tabs are not allowed**; 2 spaces is the usual convention):

```yaml
gateway:
  id: gw-01
  location:
    building: A
    floor: 2
```

**Sequences** (lists, like JSON arrays) use a dash `-` followed by a space:

```yaml
sensors:
  - temp-sensor-01
  - hum-sensor-02
```

Sequences of mappings are quite common (e.g., a list of devices):

```yaml
devices:
  - device_id: temp-sensor-01
    type: TemperatureSensor
    sampling_period_ms: 1000
  - device_id: smart-light-03
    type: SmartLight
```

**Scalars** are typed automatically:

```yaml
name: temp-sensor-01      # string (quotes are optional)
description: "Kitchen sensor: main"   # quotes needed when the value contains ": " or starts with special chars
port: 1883                # int
threshold: 25.5           # float
enabled: true             # bool
last_value: null          # null (also ~)
```

**Comments** start with `#` and are ignored by the parser.

**Flow style**: JSON-like inline syntax is also allowed, useful for short lists or mappings:

```yaml
units: [C, F, K]
position: {lat: 44.6471, lon: 10.9252}
```

## Advanced features

**Multi-line strings**: `|` (literal) keeps newlines, `>` (folded) joins lines with spaces:

```yaml
certificate: |
  -----BEGIN CERTIFICATE-----
  MIIBszCCAVmgAwIBAgIU...
  -----END CERTIFICATE-----
description: >
  This gateway collects data from all the sensors
  of the second floor and forwards it to the cloud.
```

**Anchors (`&`) and aliases (`*`)** help avoid repetition. With the merge key `<<` a mapping can inherit default values:

```yaml
defaults: &sensor_defaults
  sampling_period_ms: 1000
  unit: C
  enabled: true

sensors:
  - device_id: temp-sensor-01
    <<: *sensor_defaults          # inherits all default values
  - device_id: temp-sensor-02
    <<: *sensor_defaults
    sampling_period_ms: 5000      # overrides a single value
```

**Multiple documents** in the same file are separated by `---` (widely used, for example, in Kubernetes manifests).

## Where YAML is used in IoT

YAML is commonly used for data that **configures** systems and is edited by people, for example:

- **Device and gateway configuration**: broker address, topics, sampling periods, thresholds, list of sensors.
- **Container orchestration**: `docker-compose.yml` (see the course Docker laboratory) and Kubernetes manifests.
- **Edge/IoT platforms**: Home Assistant, ESPHome, Eclipse Ditto/Hono deployments, Node-RED settings.
- **CI/CD pipelines**: GitHub Actions workflows (`.github/workflows/*.yml`), GitLab CI.
- **API specifications**: OpenAPI documents are typically written in YAML.

Example: a `docker-compose.yml` for an IoT stack:

```yaml
services:
  mqtt-broker:
    image: eclipse-mosquitto:2.0
    ports:
      - "1883:1883"
  http-api:
    build: ./http-api
    environment:
      - MQTT_BROKER_HOST=mqtt-broker
    depends_on:
      - mqtt-broker
```

> As a general rule of thumb, YAML is usually preferred for **configuration written by humans**, while JSON is more
> common for **messages exchanged by machines**. Exceptions exist in both directions.

## Python: the PyYAML library

YAML is not part of the Python standard library. Install [PyYAML](https://pyyaml.org/):

```bash
pip install pyyaml
```

```python
import yaml
```

| Function | From | To |
|----------|------|----|
| `yaml.safe_load(stream)` | YAML string or file | Python object |
| `yaml.safe_dump(obj, stream)` | Python object | YAML file (or string if `stream` is omitted) |
| `yaml.safe_load_all(stream)` | multi-document YAML | iterator of Python objects |
| `yaml.safe_dump_all(objs, stream)` | list of Python objects | multi-document YAML |

> ⚠️ **Using `safe_load` is strongly recommended**. The generic `yaml.load()` with the unsafe loader can build arbitrary Python
> objects and execute code if the file comes from an untrusted source. `safe_load` only creates basic types
> (dict, list, str, int, float, bool, None, dates).

## Load a YAML configuration file

Given a `gateway_config.yaml` file:

```yaml
# Gateway configuration
gateway_id: gw-01
mqtt:
  host: localhost
  port: 1883
  base_topic: home/sensors
devices:
  - device_id: temp-sensor-01
    type: TemperatureSensor
    sampling_period_ms: 1000
  - device_id: hum-sensor-02
    type: HumiditySensor
    sampling_period_ms: 5000
```

```python
import yaml

with open("gateway_config.yaml", "r", encoding="utf-8") as f:
    config = yaml.safe_load(f)

print(type(config))                       # Output: <class 'dict'>
print(config["mqtt"]["host"], config["mqtt"]["port"])   # Output: localhost 1883

for device in config["devices"]:
    topic = f"{config['mqtt']['base_topic']}/{device['device_id']}"
    print(device["device_id"], device["sampling_period_ms"], topic)
```

Output:

```text
<class 'dict'>
localhost 1883
temp-sensor-01 1000 home/sensors/temp-sensor-01
hum-sensor-02 5000 home/sensors/hum-sensor-02
```

Once loaded, the configuration is a normal Python dictionary: essentially the same kind of structure you would get
from `json.load()`.

## Write a YAML file

```python
import yaml

config = {
    "gateway_id": "gw-02",
    "mqtt": {"host": "broker.example.com", "port": 8883, "tls": True},
    "devices": [
        {"device_id": "temp-sensor-05", "type": "TemperatureSensor", "sampling_period_ms": 2000},
    ],
}

with open("new_gateway_config.yaml", "w", encoding="utf-8") as f:
    yaml.safe_dump(config, f, sort_keys=False, default_flow_style=False)

print(yaml.safe_dump(config, sort_keys=False))   # without a stream it returns a string
```

Output:

```yaml
gateway_id: gw-02
mqtt:
  host: broker.example.com
  port: 8883
  tls: true
devices:
- device_id: temp-sensor-05
  type: TemperatureSensor
  sampling_period_ms: 2000
```

- `sort_keys=False` keeps the original key order (by default keys are sorted alphabetically).
- `default_flow_style=False` forces the block (indented) style.
- `allow_unicode=True` writes non-ASCII characters as they are.
- Comments are **not** preserved: `safe_load` discards them and `safe_dump` cannot write them
  (the [`ruamel.yaml`](https://pypi.org/project/ruamel.yaml/) library supports round-trip editing with comments).

## Multiple documents in one file

```python
import yaml

text = """
device_id: temp-sensor-01
type: TemperatureSensor
---
device_id: smart-light-03
type: SmartLight
"""

for doc in yaml.safe_load_all(text):
    print(doc)
```

Output:

```text
{'device_id': 'temp-sensor-01', 'type': 'TemperatureSensor'}
{'device_id': 'smart-light-03', 'type': 'SmartLight'}
```

## Converting between YAML and JSON

Since both formats share (almost) the same data model, converting is usually just "load with one, dump with the
other". A possible scenario: the configuration is written in YAML by an operator, and sent as JSON to a device through MQTT
or an HTTP API.

```python
import json
import yaml

with open("gateway_config.yaml", "r", encoding="utf-8") as f:
    config = yaml.safe_load(f)

json_config = json.dumps(config)              # YAML -> JSON (e.g., to send it over the network)
back_to_yaml = yaml.safe_dump(json.loads(json_config), sort_keys=False)   # JSON -> YAML
print(json_config)
```

## Common pitfalls

- **Tabs are not allowed** for indentation, and wrong indentation can silently change the structure.
- **Implicit typing surprises** (YAML 1.1, used by PyYAML):
  - `country: NO` → `False` (the "Norway problem"); also `yes/no/on/off` are booleans.
  - `version: 1.10` → float `1.1`; `zip: 01234` → may be parsed as an octal integer.
  - `time: 12:30` → may become a number (sexagesimal) in YAML 1.1.
  - **A common solution** is to quote values that need to remain strings: `version: "1.10"`, `country: "NO"`.
- `key:value` (without space) is a string, not a mapping: a space after `:` is needed.
- Values containing `: ` or starting with `{`, `[`, `*`, `&`, `#`, `@` usually need to be quoted.
- An empty file is loaded as `None`, not as an empty dictionary: it is worth handling this case (e.g., `config = yaml.safe_load(f) or {}`).
- A configuration can be syntactically valid and still wrong: see [Data Validation](07-validation.md).

```python
import yaml

print(yaml.safe_load("country: NO\nversion: 1.10\nenabled: on"))
# Output: {'country': False, 'version': 1.1, 'enabled': True}
print(yaml.safe_load('country: "NO"\nversion: "1.10"\nenabled: "on"'))
# Output: {'country': 'NO', 'version': '1.10', 'enabled': 'on'}
```

---

[⬅ Previous: JSON](01-json.md) · [Back to README](../README.md) · [Next: TOML ➡](03-toml.md)
