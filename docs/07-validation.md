# Data Validation

This document introduces the concept of **data validation**, explains why it is particularly important in distributed IoT systems,
and shows how to validate JSON messages and YAML configuration files in Python using **JSON Schema**.

[⬅ Previous: CBOR](06-cbor.md) · [Back to README](../README.md) · [Next: Formats Comparison ➡](08-comparison.md)

- [Why validate data](#why-validate-data)
- [Three levels of validation](#three-levels-of-validation)
- [Where to validate](#where-to-validate)
- [JSON Schema](#json-schema)
- [Main JSON Schema keywords](#main-json-schema-keywords)
- [Python: the jsonschema library](#python-the-jsonschema-library)
- [Validating JSON telemetry messages](#validating-json-telemetry-messages)
- [Reporting all the errors](#reporting-all-the-errors)
- [Validating a YAML configuration](#validating-a-yaml-configuration)
- [Writing the schema in YAML](#writing-the-schema-in-yaml)
- [Beyond JSON Schema: typed models](#beyond-json-schema-typed-models)
- [Best practices](#best-practices)

## Why validate data

In a distributed system, data often comes from **components you do not fully control**: devices with different firmware versions,
third-party services, users editing configuration files. A JSON message or a YAML file can be **perfectly valid
syntactically** and still be wrong:

```json
{"device_id": "temp-sensor-01", "value": "hot", "unit": "C"}
```

```yaml
mqtt:
  host: localhost
  port: "eighteen-eighty-three"
```

Without validation, the error may show up much later and far from its cause, for example a `TypeError` in the analytics service,
a crash when connecting to the broker, a wrong value stored in the database or shown in a dashboard.

Validation means **checking that data respects an agreed contract before using it**. Some benefits:

- **Fail fast**: errors can be detected at the boundary, with a clear message saying what is wrong and where.
- **Robustness**: a malformed message from one device is less likely to crash the whole gateway.
- **Security**: untrusted input can be rejected before reaching the application logic (e.g., huge strings, unexpected fields).
- **Documentation and contracts**: a schema is a precise, machine-readable description of the data exchanged between
  teams and services (producers and consumers can evolve independently as long as they respect it).

## Three levels of validation

| Level | Question | Example error | Tool |
|-------|----------|---------------|------|
| **Syntactic** | Is it well-formed JSON/YAML? | missing quote, trailing comma, wrong indentation | the parser (`json.loads`, `yaml.safe_load`) |
| **Structural (schema)** | Does it have the expected fields with the right types and ranges? | missing `device_id`, `value` is a string, `port` > 65535 | **JSON Schema** |
| **Semantic (business rules)** | Does it make sense in our domain? | unknown device, timestamp in the future, `min_threshold` > `max_threshold` | application code |

The first level is handled by the parsers seen in the [JSON](01-json.md#handling-invalid-json) and
[YAML](02-yaml.md) documents. This document focuses on the second level, which is usually the most reusable one.

## Where to validate

```text
   Device ──MQTT──▶ Gateway ──HTTP──▶ Cloud API ──▶ Database
                       ▲                  ▲
               validate incoming   validate request
                   messages             bodies

  config.yaml ──▶ Application startup
                       ▲
             validate config before
             starting (fail fast)
```

- **Inbound messages** (MQTT/CoAP/HTTP): messages are commonly validated on arrival; on error, a typical approach is
  to log and discard them (or reply with an error, e.g. HTTP `400 Bad Request`) instead of crashing.
- **Configuration files**: usually validated at startup; on error, the application typically stops with a clear message.
- **Outbound messages**: validating them in tests helps ensure that what you produce respects the contract.

## JSON Schema

[JSON Schema](https://json-schema.org/) is a standard vocabulary to describe the structure of JSON data.
A schema is itself a JSON document. Example: the schema of a telemetry message.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Telemetry message",
  "type": "object",
  "properties": {
    "device_id": {"type": "string", "pattern": "^[a-z0-9-]+$"},
    "value":     {"type": "number"},
    "unit":      {"type": "string", "enum": ["Cel", "%RH", "lx", "W"]},
    "timestamp": {"type": "integer", "minimum": 0}
  },
  "required": ["device_id", "value", "unit", "timestamp"],
  "additionalProperties": false
}
```

Since YAML, TOML and CBOR share (largely) the same data model as JSON (dictionaries, lists, strings, numbers...), **the same
JSON Schema can usually validate data loaded from any of these formats**: validation happens on the Python objects, after
parsing.

## Main JSON Schema keywords

| Keyword | Applies to | Meaning |
|---------|-----------|---------|
| `type` | any | `object`, `array`, `string`, `number`, `integer`, `boolean`, `null` (or a list of types) |
| `enum` / `const` | any | value must be one of a list / exactly a value |
| `properties` | object | schema of each field |
| `required` | object | list of mandatory fields |
| `additionalProperties` | object | `false` rejects unknown fields |
| `minimum` / `maximum` / `exclusiveMinimum` / `exclusiveMaximum` | number | numeric range |
| `minLength` / `maxLength` / `pattern` | string | length and regular expression |
| `format` | string | e.g. `date-time`, `email`, `ipv4`, `uri` (annotation, checked only if enabled) |
| `items` / `minItems` / `maxItems` / `uniqueItems` | array | schema of the elements and size |
| `default` / `description` | any | documentation only (not applied automatically) |
| `$ref` / `$defs` | any | reuse schema fragments |

## Python: the jsonschema library

```bash
pip install jsonschema
```

| Function / class | Use |
|------------------|-----|
| `jsonschema.validate(instance, schema)` | raises `ValidationError` at the first error |
| `Draft202012Validator(schema)` | reusable validator object (faster when validating many messages) |
| `validator.iter_errors(instance)` | iterates over **all** the errors |
| `Draft202012Validator.check_schema(schema)` | checks that the schema itself is valid |

## Validating JSON telemetry messages

```python
import json
from jsonschema import validate, ValidationError

TELEMETRY_SCHEMA = {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": {
        "device_id": {"type": "string", "pattern": "^[a-z0-9-]+$"},
        "value": {"type": "number"},
        "unit": {"type": "string", "enum": ["Cel", "%RH", "lx", "W"]},
        "timestamp": {"type": "integer", "minimum": 0},
    },
    "required": ["device_id", "value", "unit", "timestamp"],
    "additionalProperties": False,
}

def parse_telemetry(payload):
    """Returns the message as a dict, or None if it is not valid."""
    try:
        message = json.loads(payload)                 # 1. syntactic validation
        validate(message, TELEMETRY_SCHEMA)           # 2. structural validation
        return message
    except json.JSONDecodeError as e:
        print(f"Discarded: invalid JSON ({e.msg})")
    except ValidationError as e:
        path = "/".join(str(p) for p in e.absolute_path) or "(root)"
        print(f"Discarded: {path}: {e.message}")
    return None

parse_telemetry('{"device_id": "temp-sensor-01", "value": 21.5, "unit": "Cel", "timestamp": 1727535600}')
parse_telemetry('{"device_id": "temp-sensor-01", "value": "hot", "unit": "Cel", "timestamp": 1727535600}')
parse_telemetry('{"device_id": "temp-sensor-01", "value": 21.5, "unit": "C", "timestamp": 1727535600}')
parse_telemetry('{"device_id": "temp-sensor-01", "value": 21.5, "unit": "Cel"}')
parse_telemetry('{"device_id": "temp-sensor-01", "value": 21.5,')
```

Output (the first message is valid and prints nothing):

```text
Discarded: value: 'hot' is not of type 'number'
Discarded: unit: 'C' is not one of ['Cel', '%RH', 'lx', 'W']
Discarded: (root): 'timestamp' is a required property
Discarded: invalid JSON (Expecting property name enclosed in double quotes)
```

In an MQTT consumer, `parse_telemetry(msg.payload)` would be called inside the `on_message` callback: invalid messages
are logged and discarded while the consumer keeps running.

## Reporting all the errors

`validate()` stops at the first error. When validating a configuration file, it is often more useful to show **all**
the problems at once:

```python
from jsonschema import Draft202012Validator

validator = Draft202012Validator(TELEMETRY_SCHEMA)   # schema from the previous example

message = {"device_id": "Temp Sensor", "value": "hot", "unit": "C", "battery": 87}

for error in sorted(validator.iter_errors(message), key=lambda e: list(e.absolute_path)):
    path = "/".join(str(p) for p in error.absolute_path) or "(root)"
    print(f"- {path}: {error.message}")
```

Output:

```text
- (root): 'timestamp' is a required property
- (root): Additional properties are not allowed ('battery' was unexpected)
- device_id: 'Temp Sensor' does not match '^[a-z0-9-]+$'
- unit: 'C' is not one of ['Cel', '%RH', 'lx', 'W']
- value: 'hot' is not of type 'number'
```

## Validating a YAML configuration

A YAML file is loaded with `yaml.safe_load()` into normal Python dictionaries and lists, which are then validated with a
JSON Schema, in the same way as a JSON message.

Schema of the gateway configuration (nested objects, arrays, ranges and defaults):

```python
GATEWAY_CONFIG_SCHEMA = {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": {
        "gateway_id": {"type": "string", "minLength": 1},
        "mqtt": {
            "type": "object",
            "properties": {
                "host": {"type": "string"},
                "port": {"type": "integer", "minimum": 1, "maximum": 65535},
                "base_topic": {"type": "string"},
            },
            "required": ["host", "port"],
            "additionalProperties": False,
        },
        "devices": {
            "type": "array",
            "minItems": 1,
            "items": {
                "type": "object",
                "properties": {
                    "device_id": {"type": "string"},
                    "type": {"enum": ["TemperatureSensor", "HumiditySensor", "SmartLight"]},
                    "sampling_period_ms": {"type": "integer", "minimum": 100, "default": 1000},
                },
                "required": ["device_id", "type"],
            },
        },
    },
    "required": ["gateway_id", "mqtt", "devices"],
}
```

Loading and validating at startup (fail fast):

```python
import sys
import yaml
from jsonschema import Draft202012Validator

def load_config(path):
    with open(path, "r", encoding="utf-8") as f:
        config = yaml.safe_load(f) or {}          # an empty file returns None

    errors = list(Draft202012Validator(GATEWAY_CONFIG_SCHEMA).iter_errors(config))
    if errors:
        print(f"Invalid configuration file '{path}':")
        for error in errors:
            path_str = "/".join(str(p) for p in error.absolute_path) or "(root)"
            print(f"  - {path_str}: {error.message}")
        sys.exit(1)                               # do not start with a broken configuration
    return config
```

Given this (wrong) `gateway_config.yaml`:

```yaml
gateway_id: gw-01
mqtt:
  host: localhost
  port: 18830000
devices:
  - device_id: temp-sensor-01
    type: TemperatureSensor
    sampling_period_ms: 10
  - device_id: light-01
    type: Light
```

`load_config("gateway_config.yaml")` prints the following and exits:

```text
Invalid configuration file 'gateway_config.yaml':
  - mqtt/port: 18830000 is greater than the maximum of 65535
  - devices/0/sampling_period_ms: 10 is less than the minimum of 100
  - devices/1/type: 'Light' is not one of ['TemperatureSensor', 'HumiditySensor', 'SmartLight']
```

Notice how the error **path** (`devices/1/type`) helps locate the wrong line of the file quickly.

> ⚠️ Schema validation can also catch the YAML implicit typing problems described in the [YAML document](02-yaml.md#common-pitfalls):
> if `version: 1.10` is loaded as a float but the schema requires `{"type": "string"}`, the error is reported immediately.

> `default` values are **not** applied by `jsonschema`: a common approach is `device.get("sampling_period_ms", 1000)` in the code
> (or a library like pydantic, see below).

## Writing the schema in YAML

Since a schema is just data, it can be written in YAML too, which many people find more readable and which allows comments.
Keeping the schema in a separate file lets producers and consumers share the same contract:

```yaml
# telemetry.schema.yaml
$schema: https://json-schema.org/draft/2020-12/schema
title: Telemetry message
type: object
properties:
  device_id: {type: string, pattern: "^[a-z0-9-]+$"}
  value: {type: number}
  unit: {type: string, enum: [Cel, "%RH", lx, W]}
  timestamp: {type: integer, minimum: 0}
required: [device_id, value, unit, timestamp]
additionalProperties: false
```

```python
import yaml
from jsonschema import Draft202012Validator

with open("telemetry.schema.yaml", "r", encoding="utf-8") as f:
    schema = yaml.safe_load(f)

Draft202012Validator.check_schema(schema)       # the schema itself is valid
validator = Draft202012Validator(schema)

message = {"device_id": "hum-sensor-02", "value": 48.7, "unit": "%RH", "timestamp": 1727535600}
print(validator.is_valid(message))              # Output: True
```

> In YAML `%RH` needs to be quoted because `%` at the beginning of a value is a reserved character.

## Beyond JSON Schema: typed models

JSON Schema is language-independent and well suited as a **shared contract** between services written in different languages.
Inside a Python application it is also common to validate data by converting it into **typed objects**:

- [`pydantic`](https://docs.pydantic.dev/): define a class with type hints; it validates, converts types and applies
  defaults (widely used by FastAPI). It can also export a JSON Schema from the model.
- `dataclasses` + manual checks: no dependencies, good for small cases.

```python
from pydantic import BaseModel, Field   # pip install pydantic

class Telemetry(BaseModel):
    device_id: str
    value: float
    unit: str
    timestamp: int = Field(ge=0)

m = Telemetry.model_validate_json('{"device_id": "temp-sensor-01", "value": 21.5, "unit": "Cel", "timestamp": 1727535600}')
print(m.value)   # Output: 21.5
```

## Best practices

Some commonly recommended practices:

- Validating **at the boundaries** of each component (incoming messages, API requests, configuration loading).
- Keeping the three levels separate: parse → validate structure → check business rules.
- Considering `additionalProperties: false` for configurations (it catches typos like `sampling_perod_ms`), while being more tolerant
  with messages if producers may add new optional fields over time.
- **Versioning** schemas and messages (e.g., a `"schema_version": 2` field or a versioned topic/URL) so that they
  can evolve without breaking existing consumers.
- Keeping schemas in **separate files** shared between producers and consumers, and using them in automated tests.
- Reporting errors with the **path** of the wrong field: it can save a lot of debugging time.

---

[⬅ Previous: CBOR](06-cbor.md) · [Back to README](../README.md) · [Next: Formats Comparison ➡](08-comparison.md)
