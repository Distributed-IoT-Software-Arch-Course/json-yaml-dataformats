# Data Representation Formats: JSON, YAML & more

This repository is an overview of some of the most common **data representation formats**: how they work, what they
are typically used for and how to read and write them in **Python**.

The focus and the examples are designed for **Internet of Things (IoT)** and **Cyber-Physical Systems (CPS)**, where the
way data is represented plays a key role: sensors, actuators, gateways, MQTT/CoAP messages, telemetry and HTTP APIs.
At the same time, the repository also covers formats that are mainly used for other purposes, such as
**configuration files**, **logs and datasets** and **application settings**, which are part of almost any software
system.

## Why data representation matters in IoT and CPS

IoT and Cyber-Physical Systems connect the **physical world** (sensors measuring temperature, energy, position;
actuators switching lights or moving machines) with **software** running on devices, gateways and the cloud.
Every measurement, command, event and configuration crossing this boundary needs to be represented in a way that all
the involved components can interpret correctly: a wrong unit, an ambiguous timestamp or a misread field can lead to
wrong decisions with real effects on the physical environment.

These systems are also usually made of very heterogeneous components: firmware written in C on a microcontroller, a
gateway in Python, a cloud service in Java or Go, a dashboard in JavaScript. To work together they need to agree on
**how data is represented** when it is stored in a file or sent over the network (*serialization*).
How data is represented therefore commonly affects interoperability, efficiency on constrained devices and networks,
maintainability of configurations and, ultimately, the reliability of the whole system.

Different formats make different trade-offs:

- **human-readable** (e.g., configuration edited by people) vs **compact and fast** (e.g., messages from battery-powered devices);
- **flexible** (each project defines its own structure) vs **standardized** (different vendors share the same fields);
- **typed and nested** structures vs **flat tables**.

Two formats cover a large part of the typical needs:

- **JSON** is commonly used for data that **moves between machines**: data structures, messages, commands and events,
  telemetry, REST APIs.
- **YAML** is commonly used for data that **configures systems** and is written by humans: device and gateway
  configuration, docker-compose files, CI pipelines, deployment manifests.

Other formats are often chosen for more specific needs: **TOML** is a frequent alternative to YAML for application
settings, **CSV** is a simple option for tabular logs and datasets, **SenML** standardizes sensor measurements and
**CBOR** is a compact binary format, similar to JSON, designed with constrained devices in mind.

## Contents

| # | Document | Type | Typical IoT use | Python library |
|---|----------|------|-----------------|----------------|
| 1 | [JSON](docs/01-json.md) | text | data structures, messages, telemetry, REST APIs | `json` (built-in) |
| 2 | [YAML](docs/02-yaml.md) | text | configuration files, docker-compose, CI/CD | `PyYAML` |
| 3 | [TOML](docs/03-toml.md) | text | application settings, `pyproject.toml` | `tomllib` (built-in), `tomli-w` |
| 4 | [CSV](docs/04-csv.md) | text | telemetry logs, datasets, bulk import/export | `csv` (built-in) |
| 5 | [SenML](docs/05-senml.md) | JSON / CBOR | standard sensor measurements (RFC 8428) | `json` / `cbor2` |
| 6 | [CBOR](docs/06-cbor.md) | binary | constrained devices, CoAP payloads (RFC 8949) | `cbor2` |
| 7 | [Data Validation](docs/07-validation.md) | concept | validating JSON messages and YAML configs with JSON Schema | `jsonschema` |
| 8 | [Formats Comparison](docs/08-comparison.md) | summary | same data in every format, decision guide, cheat sheet | – |

Every format document follows the same structure:

1. what the format is and its specification;
2. syntax with IoT examples;
3. where it is typically used;
4. Python examples: reading and writing files, converting strings/bytes to Python objects and back;
5. common pitfalls.

## Suggested reading order

- [JSON](docs/01-json.md): a good starting point, since it is the format most frequently used in the course laboratories
  for messages and APIs.
- [YAML](docs/02-yaml.md): the format usually adopted for configuration files, such as `docker-compose.yml`.
- [TOML](docs/03-toml.md): a common alternative to YAML for application and project settings.
- [CSV](docs/04-csv.md): a simple option for storing telemetry logs and datasets for analysis.
- [SenML](docs/05-senml.md): an example of how an IoT standard represents sensor measurements in an interoperable way.
- [CBOR](docs/06-cbor.md): a binary format often used when message size and parsing cost matter.
- [Data Validation](docs/07-validation.md): how to make components more robust against wrong or unexpected data.
- [Formats Comparison](docs/08-comparison.md): a quick reference to compare the formats and choose among them.

## References

- JSON – [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259), [json.org](https://www.json.org/)
- YAML – [yaml.org](https://yaml.org/), [PyYAML documentation](https://pyyaml.org/wiki/PyYAMLDocumentation)
- TOML – [toml.io](https://toml.io/), [Python tomllib module](https://docs.python.org/3/library/tomllib.html)
- CSV – [RFC 4180](https://www.rfc-editor.org/rfc/rfc4180), [Python csv module](https://docs.python.org/3/library/csv.html)
- SenML – [RFC 8428](https://www.rfc-editor.org/rfc/rfc8428), [IANA SenML registry](https://www.iana.org/assignments/senml/senml.xhtml)
- CBOR – [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949), [cbor.io](https://cbor.io/), [cbor.me](https://cbor.me/) (online decoder)
- JSON Schema – [json-schema.org](https://json-schema.org/), [python-jsonschema](https://python-jsonschema.readthedocs.io/)
