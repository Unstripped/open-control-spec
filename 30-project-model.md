---
title: Project Model Topology & Device Catalog Formats
section: 30-project-model
version: 0.1.0
status: draft
circumvention: false
---

# Project Model Topology & Device Catalog Formats

## 1. Overview & Architectural Role

When an open-source programming client packages and uploads an automation project to a URC Total Control MRX network controller (e.g. MRX-15, MRX-30), it provides high-level system topology, zone routing maps, and device control catalogs to the controller's runtime daemons via three structured JSON configuration files installed in `/remote/Store/data/`:

1. **`mcs.dat` (Connected Device & Zone Command Model):**
   Specifies the controlled device inventory, distinguishing standard single-zone devices from multi-zone audio distribution equipment (such as HDA matrix amplifiers), and defines available function and command codes per zone `[CONFIRMED]`.
2. **`mcs_info.dat` (System Topology & Station Metadata):**
   Defines the controller station identity, registered client interfaces (smartphones, tablets, touchscreens), device database classifications, driver package bindings, download timestamps, and UI dialog targets `[CONFIRMED]`.
3. **`trf_dev_db.json` (Z-Wave Subsystem Topology):**
   Maintains runtime gateway descriptors and end-device structures for external RF/Z-Wave sub-controllers `[CONFIRMED]`.

These files are transferred during project programming as staged artifacts inside `/remote/Store/updateroot/` using model- and checksum-tagged filenames, and are committed to `/remote/Store/data/` during upload activation `[CONFIRMED]`.

---

## 2. Formatting & Serialization Constraints

All JSON documents described in this specification MUST adhere to standard ECMA-404 JSON syntax and the following serialization conventions:

1. **Encoding:** Files MUST be encoded in UTF-8 without byte-order marks (`BOM`) `[CONFIRMED]`.
2. **Whitespace:** The controller serialization emitters produce compact JSON without extra whitespace around separators (matching `,` and `:` separators) and without indentation `[CONFIRMED]`.
3. **Newlines:**
   - `mcs.dat` and `mcs_info.dat` are emitted without a trailing newline `[CONFIRMED]`.
   - `trf_dev_db.json` is emitted with a single trailing ASCII newline (`0x0A`) `[CONFIRMED]`.
4. **Key Ordering:** While JSON object keys are technically unordered, keys MUST be consistent with documented schemas to maintain deterministic token generation during manifest verification `[STATIC]`.

---

## 3. Connected Device Model (`mcs.dat`)

The `mcs.dat` file models all controllable hardware endpoints and maps each device to its supported command set and audio zone channels.

### 3.1 Root Structure

| Field | Type | Required | Allowed Values / Format | Description |
|---|---|---|---|---|
| `con_devs` | Array | **REQUIRED** | List of `ConnectedDevice` objects | Ordered collection of all devices controllable by the system `[CONFIRMED]`. |
| `editor_ver` | String | **REQUIRED** | String starting with `CCP_BIN_VER_STR` | Version header string identifying programming software build `[CONFIRMED]`. |
| `editor_type`| String | **REQUIRED** | String (e.g. `"URC Accelerator 2"`) | Identifier for programming software lineage `[CONFIRMED]`. |

### 3.2 `ConnectedDevice` Object

Each element in `con_devs` defines a single logical controlled device:

| Field | Type | Required | Allowed Values | Description |
|---|---|---|---|---|
| `id` | Integer | **REQUIRED** | Positive integer (`1..65535`) | Unique object identifier. MUST match `obj_id` in `mcs_info.dat["devices"]` and `DnData.dn` `[CONFIRMED]`. |
| `name` | String | **REQUIRED** | Non-empty UTF-8 string | User-assigned device label (e.g. `"Living Room TV"`, `"HDA-8100"`) `[CONFIRMED]`. |
| `db` | Object | **REQUIRED** | `DeviceDatabase` object | Database command capabilities and zone configuration `[CONFIRMED]`. |

### 3.3 `DeviceDatabase` Object

| Field | Type | Required | Allowed Values | Description |
|---|---|---|---|---|
| `type` | Integer | **REQUIRED** | `0` or `1` | Device classification code `[CONFIRMED]`:<br>- `0`: Standard single-zone endpoint (AV receiver, display, cable box, media streamer).<br>- `1`: Multi-zone audio distribution matrix amplifier (e.g. HDA amplifier). |
| `builtins` | Array | **REQUIRED** | Array of objects (typically `[]`) | System built-in command mappings `[CONFIRMED]`. Empty when standard functions suffice. |
| `zones` | Array | **REQUIRED** | Array of `ZoneDescriptor` objects | Array of zone command sets `[CONFIRMED]`. |

#### Zone Cardinality Constraints
- For standard devices (`type == 0`), `zones` MUST contain exactly 1 element with `idx: 0` `[CONFIRMED]`.
- For multi-zone audio devices (`type == 1`), `zones` MUST contain $N$ elements representing physical amplified/preamp zone channels, indexed sequentially from `0` to $N-1$ (e.g. 24 elements for a 24-zone HDA matrix) `[CONFIRMED]`.

### 3.4 `ZoneDescriptor` & `FunctionDescriptor` Objects

A `ZoneDescriptor` contains:

| Field | Type | Required | Allowed Values | Description |
|---|---|---|---|---|
| `idx` | Integer | **REQUIRED** | `0..N-1` | 0-based zone channel index `[CONFIRMED]`. |
| `funcs` | Array | **REQUIRED** | Array of `FunctionDescriptor` | Set of discrete commands available on this zone `[CONFIRMED]`. |

Each `FunctionDescriptor` object represents an individual executable command or IR/IP function:

| Field | Type | Required | Allowed Values | Description |
|---|---|---|---|---|
| `id` | Integer | **REQUIRED** | Positive integer (`1..65535`) | Numeric command opcode/ID in the system database `[CONFIRMED]`. |
| `name` | String | **REQUIRED** | Non-empty UTF-8 string | Command label (e.g. `"Vol+"`, `"Vol-"`, `"Mute"`, `"Power"`, `"Play"`) `[CONFIRMED]`. |

Common observed standard function IDs:
- Volume: `275` (`Vol+`), `276` (`Vol-`), `277` (`Mute`), `552` (`Mute On`), `553` (`Mute Off`).
- Transport: `288` (`Play`), `289` (`Stop`), `291` (`Pause`), `279` (`CH+`), `280` (`CH-`).
- Power: `530` (`Power`), `273` (`Power On`), `274` (`Power Off`).
- Audio EQ: `2049` (`Treble+`), `2050` (`Treble-`), `2051` (`Bass+`), `2052` (`Bass-`).

---

## 4. System Topology & Metadata (`mcs_info.dat`)

The `mcs_info.dat` file catalogues system topology: primary base stations, registered user interfaces, controlled device classifications, and software build flags.

### 4.1 Root Structure

| Field | Type | Required | Description |
|---|---|---|---|
| `bstations` | Array | **REQUIRED** | List of base station controller descriptors `[CONFIRMED]`. |
| `clients` | Array | **REQUIRED** | List of user client interface descriptors `[CONFIRMED]`. |
| `devices` | Array | **REQUIRED** | List of controlled device descriptors `[CONFIRMED]`. |
| `editor_ver` | String | **REQUIRED** | Compiler version string (`"CCP_BIN_VER_STR..."`) `[CONFIRMED]`. |
| `editor_type`| String | **REQUIRED** | Programming software suite name (e.g. `"URC Accelerator 3"`) `[CONFIRMED]`. |
| `editor_name`| String | **REQUIRED** | Software product label (e.g. `"Accelerator 3 -"`) `[CONFIRMED]`. |
| `download_date`| String | **REQUIRED** | Timestamp in format `"YYYY-MM-DD HH:MM:SS"` `[CONFIRMED]`. |
| `file_indicator`| Integer | **REQUIRED** | 32-bit integer timestamp/revision identifier `[CONFIRMED]`. |
| `special_jump`| Array | **REQUIRED** | List of system UI dialog page mapping objects `[CONFIRMED]`. |
| `unified_reset`| Integer | **OPTIONAL** | Administrative reset flag (`0` or `1`). Default `0` `[CONFIRMED]`. |

### 4.2 `bstations` Array

Each element defines a system base station controller:

| Field | Type | Required | Description |
|---|---|---|---|
| `obj_id` | Integer | **REQUIRED** | Base station object ID (master controller is typically `1`) `[CONFIRMED]`. |
| `name` | String | **REQUIRED** | User-visible station name (e.g. `"MRX-15 Master"`) `[CONFIRMED]`. |
| `model` | String | **REQUIRED** | Hardware model code (e.g. `"MRX-15"`, `"MRX-30"`) `[CONFIRMED]`. |

### 4.3 `clients` Array

Each element defines an authorized client interface (mobile device or dedicated hardware remote/keypad):

| Field | Type | Required | Description |
|---|---|---|---|
| `obj_id` | Integer | **REQUIRED** | Unique client interface object ID `[CONFIRMED]`. |
| `name` | String | **REQUIRED** | Interface name (e.g. `"iPhone Mobile"`, `"Living Room Keypad"`) `[CONFIRMED]`. |
| `model` | String | **REQUIRED** | Interface profile code (`"IPhoneV2"`, `"IPadV2"`, `"Android Tablet"`, `"TKP-7600"`, `"TDC-7100"`, `"TRC-1080"`) `[CONFIRMED]`. |

### 4.4 `devices` Array

Each element defines a controlled peripheral device:

| Field | Type | Required | Description |
|---|---|---|---|
| `obj_id` | Integer | **REQUIRED** | Device ID matching `mcs.dat["con_devs"][].id` `[CONFIRMED]`. |
| `name` | String | **REQUIRED** | Device name matching `mcs.dat["con_devs"][].name` `[CONFIRMED]`. |
| `model` | String | **REQUIRED** | Hardware model if native URC hardware (e.g. `"HDA-8100"`), or `"DBMODEL"` if selected from the generic AV database `[CONFIRMED]`. |
| `db` | Object | **CONDITIONAL** | Database categorization. REQUIRED if `model == "DBMODEL"`, absent otherwise `[CONFIRMED]`. |
| `twoway` | Object | **OPTIONAL** | Two-way module driver binding. Present when device uses a smart driver `[CONFIRMED]`. |

#### `db` Object Fields
- `type` (String): Control transport (`"IR"` or `"IP"`) `[CONFIRMED]`.
- `category` (String): Device category (e.g. `"TV"`, `"Cable"`, `"AUX"`, `"AVR's and Amplifiers "`) `[CONFIRMED]`.
- `brand` (String): Equipment manufacturer (e.g. `"Apple"`, `"Comcast"`, `"Sony"`, `"LG"`) `[CONFIRMED]`.
- `model` (String): Equipment model name in the database (e.g. `"APPLE TV(4GEN)"`, `"XFINITY"`) `[CONFIRMED]`.

#### `twoway` Object Fields
- `vfd` (String): Filename of the `.vfd` driver archive installed in `/remote/Store/data/TwoWay/` (e.g. `"SonosV2.vfd"`) `[CONFIRMED]`.

### 4.5 `special_jump` Array

The `special_jump` array contains single-key dictionary entries that map standard system dialog screens to their numeric page target IDs:

```json
[
  {"Please Wait": 10},
  {"Off Site Option": 3},
  {"Sleep Timer": 3},
  {"HDA Room EQ": 1}
]
```

### 4.6 `unified_reset` Behavior

If `unified_reset` is present and set to integer `1` in `mcs_info.dat`, the controller's download commit handler deletes `/remote/Store/App/preference.db` during upload activation, forcing all UI daemons to rebuild preference states on reboot `[CONFIRMED]`. If omitted or `0`, existing preferences are retained `[CONFIRMED]`.

---

## 5. Z-Wave Gateway & Device Database (`trf_dev_db.json`)

The `trf_dev_db.json` file is maintained by the controller automation core to describe Z-Wave gateway basestations (TRF-ZW10 / TRF-ZW20) and associated end devices.

### 5.1 Structure

```json
{
  "trfzw_10_dev": null,
  "trfzw_end_dev": null
}
```

1. **Standalone Systems Without Z-Wave:**
   When no Z-Wave gateway hardware is part of the project, both `trfzw_10_dev` and `trfzw_end_dev` MUST be set to `null` `[CONFIRMED]`.
2. **Systems With Z-Wave Gateways:**
   When gateways are configured:
   - `trfzw_10_dev` is an array of gateway descriptors containing `trfzw_subsys_id`, `trfzw_subsys_name`, and `trfzw_10_mac`, terminated by a virtual entry with `trfzw_subsys_id: 10000` and `trfzw_subsys_name: "CommonTRFZW10"` `[CONFIRMED]`.
   - `trfzw_end_dev` is an array of end-device descriptor objects mapping device IDs, room IDs, and endpoint configurations `[CONFIRMED]`.

---

## 6. Staging & Activation Workflow

During project deployment:
1. The programming tool formats and writes `mcs.dat` and `mcs_info.dat` `[CONFIRMED]`.
2. The programmer computes the 32-bit rotating checksum of each file (per `spec/20-upload-manifest.md`) `[CONFIRMED]`.
3. The files are uploaded to `/remote/Store/updateroot/` named as:
   - `M{MODEL}_M{MAC}0000_C{CHKSUM}_mcs.dat`
   - `M{MODEL}_M{MAC}0000_C{CHKSUM}_mcs_info.dat`
4. The programmer includes corresponding records in the controller's `.info` manifest `[CONFIRMED]`.
5. Upon triggering upload activation (`END_DOWNLOAD`), the controller relocates the files to `/remote/Store/data/mcs.dat` and `/remote/Store/data/mcs_info.dat` `[CONFIRMED]`.
6. On startup, the controller core initializes `trf_dev_db.json` in `/remote/Store/data/` `[CONFIRMED]`.

---

## 7. Error Behavior

1. **Missing `mcs.dat`:**
   `rmsmcsman.exe` logs `[SetEditorVersion] Read db file error...` and fails to report editor version or connected devices `[CONFIRMED]`.
2. **Missing `mcs_info.dat`:**
   During upload activation, `BS.exe` logs `[CheckUnifiedPreferencedbDelete] mcs_info data is not exists...` and skips preference database cleanup `[CONFIRMED]`.
3. **JSON Syntax Error:**
   If either file contains malformed JSON syntax, the parser logs `Json parse error...` and aborts processing that configuration file `[CONFIRMED]`.
4. **Device ID Inconsistency:**
   If an ID exists in `mcs.dat["con_devs"]` but is missing from `mcs_info.dat["devices"]`, zone and model mapping fail to resolve for that device `[STATIC]`.

---

## 8. Test Vectors

### 8.1 Minimal Project Test Vector

The following synthesized files constitute a valid minimal project defining 1 controller, 1 mobile client, and 1 television display.

#### `mcs.dat` (Single line, compact, 252 bytes)
```json
{"con_devs":[{"id":2,"name":"Living Room TV","db":{"type":0,"builtins":[],"zones":[{"idx":0,"funcs":[{"id":530,"name":"Power"},{"id":275,"name":"Vol+"},{"id":276,"name":"Vol-"},{"id":277,"name":"Mute"}]}]}}],"editor_ver":"CCP_BIN_VER_STR3.02.0628.4     ","editor_type":"URC Accelerator 2"}
```
- Expected rotating checksum (per `spec/20-upload-manifest.md`): `0x000B2A01` (`000B2A01`) `[CONFIRMED]`.

#### `mcs_info.dat` (Single line, compact, 479 bytes)
```json
{"bstations":[{"obj_id":1,"name":"Master MRX-15","model":"MRX-15"}],"clients":[{"obj_id":5,"name":"Mobile Client","model":"IPhoneV2"}],"devices":[{"obj_id":2,"name":"Living Room TV","model":"DBMODEL","db":{"type":"IR","category":"TV","brand":"Generic","model":"GENERIC_TV"}}],"editor_ver":"CCP_BIN_VER_STR3.02.0628.4     ","editor_type":"URC Accelerator 3","editor_name":"Accelerator 3 -","download_date":"2026-11-02 12:00:00","file_indicator":1730548800,"special_jump":[{"Please Wait":10},{"Off Site Option":3},{"Sleep Timer":3},{"HDA Room EQ":1}]}
```
- Expected rotating checksum: `0x0014FABA` (`0014FABA`) `[CONFIRMED]`.

#### `trf_dev_db.json` (Compact with newline, 43 bytes)
```json
{"trfzw_10_dev":null,"trfzw_end_dev":null}
```
- Expected rotating checksum: `0x0001DAEB` (`0001DAEB`) `[CONFIRMED]`.

---

## 9. Open Items

1. **Non-empty `builtins` schema:** Identify real-world device drivers that populate non-empty `con_devs[].db.builtins` arrays (observed empty across all devices in reference dump).
2. **Multi-station master/slave topology:** Document `bstations` array cardinality and network routing rules when multiple MRX controllers operate in a distributed multi-controller system.
