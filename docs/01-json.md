# JSON – JavaScript Object Notation

This document introduces JSON and Python's built-in `json` module using examples from the IoT world
(sensors, actuators, gateways and smart homes).

[⬅ Back to README](../README.md) · [Next: YAML ➡](02-yaml.md)

- [What is JSON](#what-is-json)
- [Basic components of JSON](#basic-components-of-json)
- [Where JSON is used in IoT](#where-json-is-used-in-iot)
- [Python: the json module](#python-the-json-module)
- [Load JSON data from a file](#load-json-data-from-a-file)
- [Dump data to a JSON file](#dump-data-to-a-json-file)
- [JSON strings and dictionaries](#json-strings-and-dictionaries)
- [Formatting options](#formatting-options)
- [Serializing custom objects and dates](#serializing-custom-objects-and-dates)
- [Handling invalid JSON](#handling-invalid-json)
- [Common pitfalls](#common-pitfalls)

## What is JSON

JSON (JavaScript Object Notation) is a lightweight, text-based data-interchange format that is easy for humans to read
and write, and easy for machines to parse and generate. It is language-independent, which makes it well suited for
exchanging data between different systems.

- Specification: [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259) and [json.org](https://www.json.org/)
- Media type: `application/json`
- File extension: `.json`

This is one of the reasons why JSON is widely used in IoT: a temperature sensor, a gateway, a cloud platform and a mobile
app can be written in different languages and run on very different hardware, but they can usually all read and write
the same JSON messages.

An example of a JSON document describing a single temperature measurement:

```json
{
  "device_id": "temp-sensor-01",
  "value": 21.5,
  "unit": "C"
}
```

## Basic components of JSON

**Objects**
- Enclosed in curly braces (`{}`).
- Contain key-value pairs separated by commas.
- Keys are strings (always in double quotes).
- Values can be any valid JSON data type (objects, arrays, strings, numbers, booleans, or null).

**Arrays**
- Enclosed in square brackets (`[]`).
- Contain ordered lists of values separated by commas.
- Values can be any valid JSON data type.

**Data types** (with IoT examples):

| JSON type | Example | Python type |
|-----------|---------|-------------|
| String    | `"device_id": "smart-light-03"` | `str` |
| Number    | `"timestamp": 1727535600000`, `"value": 48.7` | `int` / `float` |
| Boolean   | `"online": true` | `bool` (`True` / `False`) |
| Null      | `"value": null` (no measurement yet) | `None` |
| Object    | `"location": {"room": "kitchen"}` | `dict` |
| Array     | `"supported_units": ["C", "F", "K"]` | `list` |

**Characteristics of JSON**

- Human-readable: a developer can usually open a device message and understand it quickly.
- Machine-readable: devices, gateways and cloud services can easily parse and generate it.
- Lightweight: generally more compact than XML, which can be relevant for constrained devices and networks.
- Language-independent: firmware in C, a gateway in Python and a dashboard in JavaScript can share the same data.
- Hierarchical: data is organized using nested objects and arrays (e.g., a home containing rooms containing devices).

Example of a richer JSON structure describing a device, combining all the data types, nested objects and arrays:

```json
{
  "device_id": "temp-sensor-01",
  "device_type": "TemperatureSensor",
  "device_manufacturer": "Acme Inc.",
  "online": true,
  "firmware_version": "1.4.2",
  "location": {
    "room": "kitchen",
    "latitude": 44.6471,
    "longitude": 10.9252
  },
  "supported_units": ["C", "F", "K"],
  "last_measurement": {
    "value": 21.5,
    "unit": "C",
    "timestamp": 1727535600000
  },
  "error": null
}
```

## Where JSON is used in IoT

JSON is often the default choice for data that **moves** between components. Some typical use cases:

### 1. Data structures (device descriptions, digital twins)

A gateway or a cloud platform commonly keeps a description of the connected devices: the example above shows a
possible device descriptor. The same structure can be stored in a file, a document database (e.g., MongoDB) or returned by an API.

### 2. Messages (commands and events)

Devices and applications exchange messages through protocols like MQTT or CoAP. The payload is often a small JSON object.

A **command** sent to an actuator on topic `home/kitchen/light-03/command`:

```json
{"command_id": "c-8812", "action": "SWITCH", "payload": "ON", "timestamp": 1727535600000}
```

An **event** published by a device on topic `home/kitchen/door-01/event`:

```json
{"device_id": "door-01", "event": "DOOR_OPENED", "timestamp": 1727535612000}
```

### 3. Telemetry (measurements)

Sensors commonly publish measurements periodically. To reduce overhead, multiple samples can be grouped in a single batch:

```json
{
  "device_id": "temp-sensor-01",
  "unit": "C",
  "samples": [
    {"t": 1727535600000, "v": 21.5},
    {"t": 1727535601000, "v": 21.7},
    {"t": 1727535602000, "v": 21.4}
  ]
}
```

> Projects often define their own telemetry structure. [SenML](05-senml.md) is a standard JSON (and CBOR)
> representation for sensor measurements that can help address this interoperability problem.

### 4. REST / HTTP APIs

JSON is the most commonly used format for HTTP APIs. Typically, the client sends a request body and declares its format with the
`Content-Type` header; the server replies with a JSON body.

```http
POST /api/v1/devices HTTP/1.1
Host: iot.example.com
Content-Type: application/json

{"device_id": "hum-sensor-02", "device_type": "HumiditySensor", "room": "bathroom"}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/v1/devices/hum-sensor-02

{"device_id": "hum-sensor-02", "device_type": "HumiditySensor", "room": "bathroom", "created_at": "2024-09-28T15:00:00Z"}
```

## Python: the json module

Python provides built-in support for JSON through the `json` module (no installation required):

```python
import json
```

The module has four main functions. The ones ending with **s** work with **s**trings, the others with files:

| Function | From | To |
|----------|------|----|
| `json.load(f)`   | JSON file   | Python object |
| `json.dump(obj, f)` | Python object | JSON file |
| `json.loads(s)`  | JSON string | Python object |
| `json.dumps(obj)` | Python object | JSON string |

## Load JSON data from a file

Given a `devices_config.json` file:

```json
{
  "home_id": "SH001",
  "devices": [
    {"device_id": "temp-sensor-01", "device_type": "TemperatureSensor", "sampling_period_ms": 1000},
    {"device_id": "hum-sensor-02", "device_type": "HumiditySensor", "sampling_period_ms": 5000},
    {"device_id": "smart-light-03", "device_type": "SmartLight"}
  ]
}
```

`json.load()` reads the file and converts it into Python objects (dictionaries and lists):

```python
import json

with open("devices_config.json", "r", encoding="utf-8") as f:
    config = json.load(f)

print(config["home_id"])                     # Output: SH001
for device in config["devices"]:
    # .get() returns a default value when an optional key is missing
    print(device["device_id"], device.get("sampling_period_ms", "n/a"))
```

Output:

```text
SH001
temp-sensor-01 1000
hum-sensor-02 5000
smart-light-03 n/a
```

## Dump data to a JSON file

`json.dump()` writes Python objects to a file as JSON, for example to save the measurements collected by a sensor:

```python
import json

measurements = [
    {"device_id": "temp-sensor-01", "value": 21.5, "unit": "C", "timestamp": 1727535600000},
    {"device_id": "temp-sensor-01", "value": 21.7, "unit": "C", "timestamp": 1727535601000},
    {"device_id": "temp-sensor-01", "value": 21.4, "unit": "C", "timestamp": 1727535602000}
]

with open("temperature_log.json", "w", encoding="utf-8") as f:
    json.dump(measurements, f, indent=2)
```

The optional `indent` parameter produces a nicely formatted (human-readable) file.

## JSON strings and dictionaries

`json.dumps()` and `json.loads()` work with strings instead of files. This is typically what happens when a device sends a message
over the network: data is converted into a JSON string (and then into bytes) before being transmitted, and converted
back into a dictionary by the receiver.

From dictionary to JSON string (e.g., a sensor preparing a message to publish):

```python
import json

measurement = {
    "device_id": "hum-sensor-02",
    "value": 48.7,
    "unit": "%RH",
    "timestamp": 1727535600000
}

json_message = json.dumps(measurement)
print(json_message)
# Output: {"device_id": "hum-sensor-02", "value": 48.7, "unit": "%RH", "timestamp": 1727535600000}

payload = json_message.encode("utf-8")   # bytes, ready to be sent with MQTT/CoAP/HTTP
```

From JSON string to dictionary (e.g., a smart light receiving a command):

```python
import json

json_command = '{"device_id": "smart-light-03", "action": "SWITCH", "payload": "ON"}'
command = json.loads(json_command)

print(command["device_id"])     # Output: smart-light-03
print(command["action"])        # Output: SWITCH
print(command["payload"])       # Output: ON
```

> `json.loads()` also accepts `bytes` directly, so a received MQTT payload (`msg.payload`) can be passed as it is.

## Formatting options

```python
import json

data = {"value": 21.5, "device_id": "temp-sensor-01", "unit": "°C"}

print(json.dumps(data, indent=2))                    # pretty print (files, logs, debugging)
print(json.dumps(data, sort_keys=True))              # stable key order (diffs, tests, hashing)
print(json.dumps(data, separators=(",", ":")))       # compact: no spaces (smaller network payloads)
print(json.dumps(data, ensure_ascii=False))          # keep non-ASCII chars (°) instead of \u00b0
```

Output:

```text
{
  "value": 21.5,
  "device_id": "temp-sensor-01",
  "unit": "\u00b0C"
}
{"device_id": "temp-sensor-01", "unit": "\u00b0C", "value": 21.5}
{"value":21.5,"device_id":"temp-sensor-01","unit":"\u00b0C"}
{"value": 21.5, "device_id": "temp-sensor-01", "unit": "°C"}
```

## Serializing custom objects and dates

`json.dumps()` only knows the basic Python types (`dict`, `list`, `str`, `int`, `float`, `bool`, `None`).
Objects of your classes and `datetime` values raise a `TypeError`. There are two common solutions.

**1. Convert the object to a dictionary explicitly** (usually a clear option for your own classes):

```python
import json

class Measurement:
    def __init__(self, device_id, value, unit, timestamp):
        self.device_id = device_id
        self.value = value
        self.unit = unit
        self.timestamp = timestamp

    def to_dict(self):
        return {"device_id": self.device_id, "value": self.value,
                "unit": self.unit, "timestamp": self.timestamp}

    @classmethod
    def from_dict(cls, data):
        return cls(data["device_id"], data["value"], data["unit"], data["timestamp"])

m = Measurement("temp-sensor-01", 21.5, "C", 1727535600000)
message = json.dumps(m.to_dict())              # object -> JSON
received = Measurement.from_dict(json.loads(message))   # JSON -> object
print(received.value)                          # Output: 21.5
```

> With dataclasses you can use `dataclasses.asdict(obj)` instead of writing `to_dict()` by hand.

**2. Use the `default` parameter** to tell `json` how to convert unknown types (handy for `datetime`):

```python
import json
from datetime import datetime, timezone

def json_default(obj):
    if isinstance(obj, datetime):
        return obj.isoformat()          # ISO 8601 string, e.g. 2024-09-28T15:00:00+00:00
    raise TypeError(f"Type {type(obj).__name__} is not JSON serializable")

event = {"device_id": "door-01", "event": "DOOR_OPENED",
         "time": datetime(2024, 9, 28, 15, 0, 0, tzinfo=timezone.utc)}

print(json.dumps(event, default=json_default))
# Output: {"device_id": "door-01", "event": "DOOR_OPENED", "time": "2024-09-28T15:00:00+00:00"}
```

> JSON has no date type: timestamps are usually represented either as **Unix epoch numbers** (seconds or milliseconds)
> or as **ISO 8601 strings**. It is good practice to choose one convention and document it.

## Handling invalid JSON

Data coming from the network may be malformed, so it is good practice to catch `json.JSONDecodeError`:

```python
import json

raw_payload = b'{"device_id": "temp-sensor-01", "value": 21.5,}'   # trailing comma -> invalid

try:
    data = json.loads(raw_payload)
except json.JSONDecodeError as e:
    print(f"Invalid JSON message: {e.msg} (line {e.lineno}, column {e.colno})")
    # Output: Invalid JSON message: Expecting property name enclosed in double quotes (line 1, column 47)
```

A syntactically valid JSON can still have missing fields or wrong types: see [Data Validation](07-validation.md).

## Common pitfalls

- **No comments** are allowed in JSON (YAML or TOML are commonly used for configuration files that need comments).
- **No trailing commas** after the last element of an object or array.
- **Keys and strings require double quotes**: `'device'` (single quotes) is invalid.
- `True`/`False`/`None` are Python; in JSON they are `true`/`false`/`null`.
- **Dictionary keys become strings**: `json.dumps({1: "a"})` → `{"1": "a"}`.
- **Tuples become arrays** and come back as lists.
- **Floats**: `NaN` and `Infinity` are produced by Python but are not valid standard JSON (use `allow_nan=False` to
  detect them); floating-point precision may differ between languages.
- Opening files with an explicit `encoding="utf-8"` is recommended, to avoid platform-dependent defaults.

---

[⬅ Back to README](../README.md) · [Next: YAML ➡](02-yaml.md)
