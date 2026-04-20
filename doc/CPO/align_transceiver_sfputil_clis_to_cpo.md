# Sfputil and Transceiver CLIs Alignment for CPO

## Table of Contents

1. [Revision](#1-revision)
2. [Definitions/Abbreviations](#2-definitionsabbreviations)
3. [Scope](#3-scope)
4. [Overview](#4-overview)
5. [How the show CLIs work today](#5-how-the-show-clis-work-today)
6. [Changes (new code)](#6-changes-new-code)
   - [6.1 Common CPO fields (format maps)](#61-common-cpo-fields-format-maps)
   - [6.2 Vendor-specific formatting (Sfp API)](#62-vendor-specific-formatting-sfp-api)

---

## 1. Revision

| Rev | Date | Author | Change Description                                                                          |
| --- | ---- | ------ | ------------------------------------------------------------------------------------------- |
| 0.1 |      |        | Initial version: CPO/ELS display alignment for `sfputil` and `show interfaces transceiver`. |

---

## 2. Definitions/Abbreviations

| Term     | Definition                                                                                                     |
| -------- | -------------------------------------------------------------------------------------------------------------- |
| CPO      | Co-Packaged Optics — integrated optics; transceiver **type** may be reported as a string `CPO`.                |
| ELS      | External Laser Source — related fields prefixed `els_*` in info/DOM/status dicts.                              |
| CMIS     | Common Management Interface Specification.                                                                     |
| DOM      | Digital Optical Monitoring — sensors and thresholds.                                                           |
| STATE_DB | Redis State DB; `xcvrd` publishes transceiver tables consumed by `sfpshow`.                                    |

---

## 3. Scope

This HLD covers **sonic-utilities** changes to align CLIs for the CPO (Banks/ELS) concept.

**Relevant (existing) CLIs** — requirement per command:

| #   | Command                                           | Requirement |
| --- | ------------------------------------------------- | ----------- |
| 1   | `sfputil read-eeprom -p -n -o -s`                 | No change: `platform_chassis.get_sfp(physical_port)` / `Sfp` is already bank-aware for raw EEPROM access. |
| 2   | `sfputil write-eeprom -p -n -o -s -d`             | No change: same bank-aware `Sfp` path as row 1. |
| 3   | `sfputil show eeprom-hexdump -p -n`               | No change: same bank-aware `Sfp` read path (hex raw EEPROM access). |
| 4   | `sfputil show eeprom -p -d`                       | Display new **ELS** info and **ELS DOM** fields for the **CMIS CPO** case (API/EEPROM access; formatters and maps in `sfputil/main.py`). |
| 5   | `show interfaces transceiver eeprom -p -d`        | Display new **ELS** / **DOM** fields for **CMIS CPO** case (STATE_DB–backed; formatters and maps in `sfpshow`). |
| 6   | `show interfaces transceiver status -p`           | Display new **ELS** status fields for **CMIS CPO**. (STATE_DB–backed; formatters and maps in `sfpshow`) |
| 7   | `show interfaces transceiver error-status -p -hw` | Display new **CPO** error information for **CMIS CPO**. |

---

## 4. Overview

SONiC exposes transceiver data to the user through two paths:

| Path | Entry | Data source |
|------|--------|-------------|
| **HW** | `sfputil` | Platform **`Chassis.get_sfp(n)`** → **`Sfp`** APIs (`get_transceiver_info`, `get_transceiver_dom_real_value`, `get_transceiver_threshold_info`, `read_eeprom`, `write_eeprom`). |
| **State DB** | `show interfaces transceiver …` → **`sfpshow`** | **`STATE_DB`** rows `TRANSCEIVER_*` written by **`xcvrd`**. |

Typical hardware access:

```python
sfp = platform_chassis.get_sfp(physical_port)
api = sfp.get_xcvr_api()
info = sfp.get_transceiver_info()
dom = sfp.get_transceiver_dom_real_value()
overall_offset = get_overall_offset_general(api, page, offset, size) # page * PAGE_SIZE + offset
raw = sfp.read_eeprom(overall_offset, size)
```

Hardware path (`sfputil` → `Sfp` → EEPROM or APIs)
Use this flow for commands that talk to the platform without reading **STATE_DB**.

```text
  sfputil <subcommand>
           │
           ▼
  platform_chassis.get_sfp(physical_port)  ──►  Sfp
           │
           |   Sfp.read_eeprom(overall_offset, size)
           │   Sfp.write_eeprom(overall_offset, len, bytes)
           │
           │   get_transceiver_info()
           │   get_transceiver_dom_real_value()
           │   get_transceiver_threshold_info()
```

Typical DB access (Redis connector):

```python
state_db.connect(state_db.STATE_DB)
sfp_info_dict = state_db.get_all(state_db.STATE_DB, 'TRANSCEIVER_INFO|{}'.format(interface_name))
dom_info_dict = state_db.get_all(state_db.STATE_DB, 'TRANSCEIVER_DOM_SENSOR|{}'.format(first_subport)) or {}
# … same pattern for DOM_THRESHOLD, STATUS*, etc.
```

DB path (`show interfaces transceiver` → `sfpshow` → STATE_DB)
Use this flow for **`show interfaces transceiver eeprom`** and **`status`**;

```text
  click: show interfaces transceiver <subcommand>
           │
           ▼
  show/interfaces/__init__.py  spawns:
           │
           ├──  sfpshow eeprom  [-d]
           │        │
           │        ▼
           │        ├── get_all('TRANSCEIVER_INFO|<port>')
           │        ├── get_all('TRANSCEIVER_FIRMWARE_INFO|…')   (merge/sort helpers)
           │        │
           │        │  [-d]
           │        ├── get_all('TRANSCEIVER_DOM_SENSOR|<port>')
           │        └── get_all('TRANSCEIVER_DOM_THRESHOLD|<port>')  (merged into one dict)
           │
           └──  sfpshow status
           │         │
           │         ▼
           │    get_all('TRANSCEIVER_STATUS|…')
           │    get_all('TRANSCEIVER_STATUS_SW|…')
           │    get_all('TRANSCEIVER_STATUS_FLAG|…')
           │    get_all('TRANSCEIVER_DOM_FLAG|…')
```

`show interfaces transceiver error-status` → `sfputil show error-status`

```text
  show interfaces transceiver error-status  [-p]  [-hw]  [-n <namespace>]
           │
           ▼
  sudo sfputil show error-status  [-p]  [-hw]  [-n …]
           │
           ├──  default (no -hw):  STATE_DB  get_all('TRANSCEIVER_STATUS_SW|<port>')
           │                        → table: Port | Error Status  (status/error fields)
           │
           └──  -hw:  per port  →  Sfp.get_error_description()
```

```python
# Default (no -hw): STATE_DB — same keys sfputil uses to build Port | Error Status
state_db.connect(state_db.STATE_DB)
sw = state_db.get_all(state_db.STATE_DB, f"TRANSCEIVER_STATUS_SW|{port_name}")
# Typical fields: sw["status"], sw["error"]  → mapped to OK / Unplugged / description

# -hw path (per port): hardware
err = platform_chassis.get_sfp(physical_port).get_error_description()
```

The only requirement here is to ensure we have the new CPO errors and present them as part of the output.

---

## 5. How the show CLIs work today

```text
   ┌─────────────────────────────┐         ┌──────────────────────────────┐
   │  STATE_DB tables            │         │  Sfp APIs (sfputil path)     │
   │  TRANSCEIVER_INFO,          │         │  get_transceiver_info,       │
   │  DOM_SENSOR / DOM_THRESHOLD,│   OR    │  get_transceiver_dom_*       │
   │  STATUS*, …                 │         │  (same key/value dict idea)  │
   └──────────────┬──────────────┘         └───────────────┬──────────────┘
                  │                                        │
                  └────────────────┬───────────────────────┘
                                   ▼
                    Python dict:  { "els_foo": <value from source>,
                                    "els_vendor": "Vendor-A", … }
                                   │
                                   │
                                   │
                                   ▼
                    ┌──────────────────────────────────────────┐
                    │  Use the relavant format data map           │
                    │  data_map[ field_key ] → display key     │
                    │  data_map[ "els_vendor" ] → "ELS Vendor" │
                    └──────────────────┬───────────────────────┘
                                       │
                                       ▼
                    CLI text lines:    <display key> : <value from dict>
                                       ELS Vendor : Vendor-A
                                       …
```

The **map** supplies the **column header / field name** for humans; the **DB or API** supplies the **value**.

```python
# The real code is more complex but the following is a simpler version just to present the idea

DATA_MAP = {
    'model': 'Vendor PN',
    'vendor_oui': 'Vendor OUI',
    'vendor_date': 'Vendor Date Code(YYYY-MM-DD Lot)',
    'manufacturer': 'Vendor Name',
    'vendor_rev': 'Vendor Rev',
    # ...
}

# ...

sfp = platform_chassis.get_sfp(physical_port)
# Since sfp is platform object, it will return also vendor-specific fields.
info = sfp.get_transceiver_info() # Or get_all('TRANSCEIVER_INFO|<port>')

for key in info:
  print(f'{DATA_MAP[key]} : {info[key]}')
```

## 6. Changes (new code)

We need 2 main changes:
-  **Common CPO maps** Extend / create format maps for the new common CPO fields and integrate / use them as part of the flow (in case of CPO ports).
- **Vendor-specific maps** Create a way for a vendor-specific formatting, for example override the existing formatting / mapping, or create formatting for a vendor-specific fields.

### 6.1 Common CPO fields (format maps)

Extend or create format maps for the new **common** CPO fields and integrate them in the formatter flow when the transceiver is CMIS CPO (merge into the base CMIS / QSFP-DD style maps as needed).

The real code is more complex; the following is a simplified illustration:

```python
# The real code is more complex but the following is a simpler version just to present the idea

DATA_MAP = {
    'model': 'Vendor PN',
    'vendor_oui': 'Vendor OUI',
    'vendor_date': 'Vendor Date Code(YYYY-MM-DD Lot)',
    'manufacturer': 'Vendor Name',
    'vendor_rev': 'Vendor Rev',
    # ...
}

CPO_TRANSCEIVER_INFO_MAP = {
    'els_type': 'ELS type',
    'els_type_abbrv_name': 'ELS type_abbrv_name',
    'els_hardware_rev': 'ELS hardware_rev',
    'els_serial': 'ELS serial',
    # ...
}

# ...

sfp = platform_chassis.get_sfp(physical_port)
# Since sfp is platform object, it will return also vendor-specific fields.
info = sfp.get_transceiver_info() # Or get_all('TRANSCEIVER_INFO|<port>')

format_map = DATA_MAP
if sfp_type == 'CPO':
  format_map.update(CPO_TRANSCEIVER_INFO_MAP)

for key in info:
  print(f'{format_map[key]} : {info[key]}')

```

### 6.2 Vendor-specific formatting (Sfp API)

We do not add vendor-specific fields formatting in shared `sfputil` / `sfpshow` code; those maps come from **platform** `Sfp` code via a small API.

The real code is more complex; the following is a simplified illustration:

```python
# src/sonic-platform-common/sonic_platform_base/sfp_base.py:SfpBase

def get_vendor_specific_format_map(self, map_name):
        """
        Retrieves the vendor-specific format map for the given map name
        """
        return {}
```

```python
# src/sonic-py-common/sonic_py_common/sfp_format_map_names.py:

DOM_CHANNEL_MONITOR_FORMAT_MAP = "dom_channel_monitor_format_map"
DOM_MODULE_MONITOR_FORMAT_MAP = "dom_module_monitor_format_map"
DOM_VALUE_UNIT_FORMAT_MAP = "dom_value_unit_format_map"
STATUS_FORMAT_MAP = "status_format_map"
```

```python
# platform/mellanox/mlnx-platform-api/sonic_platform/sfp.py:CpoPort

CPO_DOM_CHANNEL_MONITOR_MAP = {
        'els_laser_mpd1': 'ELS Lane1LaserMPD',
        'els_tec_voltage_laser1': 'ELS Lane1TecVoltage',
        'els_tec_health_value_laser1': 'ELS Lane1TecHealth',
        'els_health_value_laser1': 'ELS Lane1LaserHealth',
        # ...
}

# ...

def get_vendor_specific_format_map(self, map_name):
        """
        Retrieves the vendor-specific format map for the given map name
        """
        if map_name == DOM_CHANNEL_MONITOR_FORMAT_MAP:
            return self.CPO_DOM_CHANNEL_MONITOR_MAP
        # elif ... 
        return {}
```

```python
# src/sonic-utilities/sfputil/main.py

CMIS_DOM_CHANNEL_MONITOR_MAP = {
    'rx1power': 'RX1Power',
    'rx2power': 'RX2Power',
    'rx3power': 'RX3Power',
    # ...
}

CPO_DOM_CHANNEL_MONITOR_MAP = {
    'els_tx1bias': 'ELS Lane1Bias',
    'els_tx1voltage': 'ELS Lane1Voltage',
    'els_tx1power': 'ELS Lane1Power',
    # ...
}

info = sfp.get_transceiver_dom_real_value()

# Adding new common CPO formatting
format_map = CMIS_DOM_CHANNEL_MONITOR_MAP
if sfp_type == 'CPO':
  format_map.update(CPO_DOM_CHANNEL_MONITOR_MAP)

# Adding new vendor-specific formatting
vendor_dom_channel_monitor_map = sfp.get_vendor_specific_format_map(DOM_CHANNEL_MONITOR_FORMAT_MAP)
format_map.update(vendor_dom_channel_monitor_map)

for key in info:
  print(f'{format_map[key]} : {info[key]}')

```

---
