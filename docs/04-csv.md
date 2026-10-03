# CSV – Comma-Separated Values

This document introduces CSV, one of the simplest formats for **tabular data** such as telemetry logs and datasets,
and shows how to read and write it with Python's built-in `csv` module.

[⬅ Previous: TOML](03-toml.md) · [Back to README](../README.md) · [Next: SenML ➡](05-senml.md)

- [What is CSV](#what-is-csv)
- [Syntax](#syntax)
- [Where CSV is used in IoT](#where-csv-is-used-in-iot)
- [Python: the csv module](#python-the-csv-module)
- [Write a CSV file](#write-a-csv-file)
- [Read a CSV file](#read-a-csv-file)
- [Append measurements to a log](#append-measurements-to-a-log)
- [From JSON messages to CSV](#from-json-messages-to-csv)
- [Analysis with pandas (optional)](#analysis-with-pandas-optional)
- [Common pitfalls](#common-pitfalls)

## What is CSV

CSV is a plain text format where each **line is a row** and **values are separated by commas**.
The first row usually contains the column names (header).

- Specification: [RFC 4180](https://www.rfc-editor.org/rfc/rfc4180) (informal, many dialects exist)
- Media type: `text/csv`
- File extension: `.csv`

```csv
device_id,timestamp,value,unit
temp-sensor-01,1727535600,21.5,Cel
temp-sensor-01,1727535610,21.7,Cel
hum-sensor-02,1727535600,48.7,%RH
```

CSV is **flat** (no nesting) and **untyped** (everything is text), but it is usually compact for tables, can be
appended line by line, and can be opened by many tools: Excel, LibreOffice, pandas, R, databases, Grafana...

## Syntax

- One record per line; fields separated by a delimiter (`,` by default; `;` or `\t` are common alternatives).
- Fields containing the delimiter, double quotes or newlines are enclosed in double quotes.
- A double quote inside a quoted field is escaped by doubling it: `""`.

```csv
device_id,location,note
temp-sensor-01,"Building A, floor 2","calibrated ""manually"""
```

## Where CSV is used in IoT

Some typical use cases:

- **Telemetry logs** on a gateway or an edge device (append one row per measurement).
- **Datasets** for data analysis and machine learning (export from a time-series database).
- **Bulk import/export** of device inventories (list of devices, serial numbers, locations).
- **Data exchange with non-developers** (spreadsheets).

CSV is usually **not** a good fit for messages between services (no structure, no types, no nesting): JSON or CBOR
are generally more suitable in that case.

## Python: the csv module

The `csv` module is built-in:

```python
import csv
```

| Class / function | Use |
|------------------|-----|
| `csv.writer(f)` / `csv.reader(f)` | rows as **lists** |
| `csv.DictWriter(f, fieldnames)` / `csv.DictReader(f)` | rows as **dictionaries** (column name → value) |

> Opening CSV files with `newline=""` is recommended: the `csv` module handles line endings by itself; without it you may get
> empty lines between rows on Windows.

## Write a CSV file

With `csv.writer` (rows as lists):

```python
import csv

rows = [
    ["temp-sensor-01", 1727535600, 21.5, "Cel"],
    ["temp-sensor-01", 1727535610, 21.7, "Cel"],
    ["hum-sensor-02", 1727535600, 48.7, "%RH"],
]

with open("telemetry.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f)
    writer.writerow(["device_id", "timestamp", "value", "unit"])   # header
    writer.writerows(rows)
```

With `csv.DictWriter` (rows as dictionaries, often a convenient choice when data comes from JSON messages):

```python
import csv

measurements = [
    {"device_id": "temp-sensor-01", "timestamp": 1727535600, "value": 21.5, "unit": "Cel"},
    {"device_id": "hum-sensor-02", "timestamp": 1727535600, "value": 48.7, "unit": "%RH"},
]

with open("telemetry.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["device_id", "timestamp", "value", "unit"])
    writer.writeheader()
    writer.writerows(measurements)
```

## Read a CSV file

```python
import csv

with open("telemetry.csv", "r", newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row)
```

Output:

```text
{'device_id': 'temp-sensor-01', 'timestamp': '1727535600', 'value': '21.5', 'unit': 'Cel'}
{'device_id': 'hum-sensor-02', 'timestamp': '1727535600', 'value': '48.7', 'unit': '%RH'}
```

⚠️ All values are **strings**, so they usually need to be converted explicitly:

```python
import csv

with open("telemetry.csv", "r", newline="", encoding="utf-8") as f:
    measurements = [
        {**row, "timestamp": int(row["timestamp"]), "value": float(row["value"])}
        for row in csv.DictReader(f)
    ]

temperatures = [m["value"] for m in measurements if m["unit"] == "Cel"]
print("Average temperature:", sum(temperatures) / len(temperatures))   # Output: Average temperature: 21.5
```

## Append measurements to a log

A gateway can store every received measurement by appending a row (`"a"` mode), writing the header only when the file
is new:

```python
import csv
import os

LOG_FILE = "telemetry_log.csv"
FIELDS = ["device_id", "timestamp", "value", "unit"]

def log_measurement(measurement):
    is_new_file = not os.path.exists(LOG_FILE)
    with open(LOG_FILE, "a", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=FIELDS, extrasaction="ignore")   # ignore extra keys
        if is_new_file:
            writer.writeheader()
        writer.writerow(measurement)

log_measurement({"device_id": "temp-sensor-01", "timestamp": 1727535600, "value": 21.5, "unit": "Cel"})
log_measurement({"device_id": "temp-sensor-01", "timestamp": 1727535610, "value": 21.7, "unit": "Cel", "rssi": -70})
```

## From JSON messages to CSV

Nested JSON typically needs to be **flattened** before writing it to CSV. For example, a [SenML](05-senml.md) pack, once resolved,
becomes one row per record:

```python
import csv
import json

payload = '''[
  {"n": "urn:dev:mac:0024befffe804ff1:temperature", "u": "Cel", "v": 21.5, "t": 1727535600},
  {"n": "urn:dev:mac:0024befffe804ff1:humidity", "u": "%RH", "v": 48.7, "t": 1727535600}
]'''

with open("senml_log.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["n", "t", "v", "u"])
    writer.writeheader()
    writer.writerows(json.loads(payload))
```

Result (`senml_log.csv`):

```csv
n,t,v,u
urn:dev:mac:0024befffe804ff1:temperature,1727535600,21.5,Cel
urn:dev:mac:0024befffe804ff1:humidity,1727535600,48.7,%RH
```

## Analysis with pandas (optional)

For data analysis, [pandas](https://pandas.pydata.org/) reads a CSV directly into a table (DataFrame) and infers types:

```python
import pandas as pd   # pip install pandas

df = pd.read_csv("telemetry.csv")
print(df.groupby("device_id")["value"].mean())
```

## Common pitfalls

- **No types**: everything is a string, numbers and booleans need to be converted manually.
- **No nesting**: nested objects/arrays need to be flattened (e.g. `location.room` → `location_room`).
- **Dialects**: delimiter (`,` vs `;`), quoting and decimal separator vary. Italian Excel, for example, uses `;` as
  delimiter and `,` as decimal separator (`21,5`). `csv.reader(f, delimiter=";")` can be used in these cases.
- Opening files with `newline=""` and an explicit `encoding="utf-8"` is recommended.
- Keeping the column order fixed is usually a good idea (e.g., `DictWriter` with an explicit `fieldnames` list).

---

[⬅ Previous: TOML](03-toml.md) · [Back to README](../README.md) · [Next: SenML ➡](05-senml.md)
