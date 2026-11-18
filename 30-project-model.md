---
title: Project Model Topology & Device Catalog Formats
section: 30-project-model
version: 0.5.0
status: draft
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

### 1.1 Relationship to Physical Rooms & LAN Client Discovery (`core_model.json`)

A critical architectural distinction exists between internal controller databases and client-facing runtime models:

1. **Internal Hardware Databases (`/remote/Store/data/`):**
   `mcs.dat` and `mcs_info.dat` define connected device hardware, matrix zones (`zones[].idx`), and controller station inventory. These files reside strictly within `/remote/Store/data/`. Because the controller's anonymous FTP server (port 21) is jailed to `/remote/Store/updateroot_anon/updateroot/`, **`mcs.dat` and `mcs_info.dat` CANNOT be read over anonymous FTP by LAN clients** `[CONFIRMED]`.

2. **Client-Facing Project Model (`/remote/Store/updateroot/`):**
   Touchscreens, mobile applications, and open-source clients discover physical rooms, assigned devices, and function codes by downloading **`core_model.json`** from `updateroot/` over anonymous FTP (port 21) `[CONFIRMED]`.
   - **Filename Pattern:** Staged as `T{TEMPLATE_ID}_A0000000000000000_C{CHKSUM}_core_model.json`. In the reference installation, `0002` is named by the iPhone (`ME207`) manifest and `0003` by the Android tablet (`ME206`) and iPad (`ME208`) manifests `[CONFIRMED]`. The programmer authors one core model per project and stages a copy under each template prefix; in the reference both copies are byte-identical `[CONFIRMED]`. A client MUST load the file named in its own model-wide manifest (`53-client-protocols` §3.4.1).
   - **Room Topology (`data.rooms[]`):** Physical rooms (`room_id`, `room_name`, and assigned devices) are defined under `data.rooms[]` and compiled into `DnData.dn` Table 10 (room table) `[CONFIRMED]`.
   - **Room Device Objects are not Connected Devices:** Each `data.rooms[].devices[].obj.id` is a project-wide UI object id. It equals the device-object table device id in that room's Table 10 record, **not** `mcs.dat` `con_devs[].id` `[CONFIRMED]`. The Table 10 device object names the connected device it controls (`32-download-database` §5.3.2). Clients see the same link as `obj.keyboard.con_dev_id` (`53-client-protocols` §3.4.3). A connected device used in several rooms appears once per room with different object ids. Multi-zone amplifiers (`db.type == 1`) are not room devices: in the reference, the 24-zone amplifier appears in no room's `devices[]` `[CONFIRMED]`.
   - **Button Codes:** Every type-33 macro step carries `code = (object_id << 16) | button_id` `[CONFIRMED]`:
     ```json
     "macro": [{"type": 33, "code": 459029}]
     ```
     `object_id` is the room device object (or the room's menu object for `menu_devs` buttons), never a connected-device id. On device pages, the vendor programmer sets `button_id` to the controlled device's function id (`funcs[].id`). Example: `459029 = (7 << 16) | 277` is Mute on reference object `7`, which controls connected device `3` `[CONFIRMED]`. The full macro step catalog is in `53-client-protocols` §3.4.4.

3. **Room-Level Commands:**
   `ROOM_POWER_OFF` (`0x000D`) and `ROOM_POWER_ON` (`0x0012`) take the room identifier defined in `data.rooms[].id` `[CONFIRMED]`. A `core_model.json` macro step of type `54` (`{"type": 54, "room": <id>}`) means Room Power Off for that room (`53-client-protocols` §3.4.4). `MACRO_TRIGGER` (`0x07DA`) carries a button `code` and no separate room or device field; see `52-control-protocol` §3.1.
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
- Source Inputs: No input-select function ids are observed in the reference `mcs.dat`. The 24-zone amplifier's zones expose `273`–`277`, `530`, `552`, `553`, `2049`–`2067`, and `3001`–`3011` only `[CONFIRMED]`. In the reference, zone source changes are steps inside controller-side room macros (`OP_ZONE_CMD` `0x35`, `32-download-database` §5.4.2) that run when a room menu tile is triggered `[CONFIRMED]` (macro contents). How a non-reference driver names its source-select functions is **unresolved**.

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
- `vfd` (String): Filename of the `.vfd` driver archive installed in `/remote/Store/data/TwoWay/` (e.g. `"ExampleModule.vfd"`) `[CONFIRMED]`.

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

## 6. Staging & Provisioning Workflow

The delivery of project models differs between runtime controller daemons and client-facing interfaces:

1. **Direct Controller Provisioning (`/remote/Store/data/`):**
   Runtime configuration files `mcs.dat`, `mcs_info.dat`, `Link.dat`, `Event.dat`, and `Alarm.dat` are provisioned directly to `/remote/Store/data/` by programming tools via administrative channels. The controller's staging installer skips these files; they are NOT installed from staged manifests and do NOT persist in `updateroot/` following a standard dealer deployment `[CONFIRMED]`.

2. **Staged Client Distribution (`/remote/Store/updateroot/`):**
   UI models (`core_model.json`, `model.json`, `ui_button_table.json`) and graphic archives are uploaded to `/remote/Store/updateroot/` formatted with template and checksum tokens:
   - `T0002_A0000000000000000_C{CHKSUM}_core_model.json` (Mobile & Template 2 touchscreens)
   - `T0003_A0000000000000000_C{CHKSUM}_core_model.json` (Template 3 high-res touchscreens)
   These staged models persist in `updateroot/` across reboots and activations, serving as the definitive discovery mechanism for anonymous LAN clients `[CONFIRMED]`.

3. **Volume values:**
   Volume fields appear in project bindings, control frames, and MQTT telemetry. Their range and units depend on the specific interface and configured device; do not assume every field is a percentage or that zero means mute. See [control](52-control-protocol.md) and [telemetry](00-system-overview.md).
---

## 7. Error Behavior

1. **Missing `mcs.dat`:**
   If `mcs.dat` is missing, the controller's remote-management agent cannot report the editor version or connected devices `[CONFIRMED]`.
2. **Missing `mcs_info.dat`:**
   If `mcs_info.dat` is missing, the controller skips the preference-database cleanup during upload activation `[CONFIRMED]`.
3. **JSON Syntax Error:**
   If either file contains malformed JSON syntax, the controller aborts processing that configuration file `[CONFIRMED]`.
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

#### `core_model.json` (Minimal Room and Device Topology)
```json
{"data":{"config":{"obj":{"system_name":"Minimal System","key":"","ddns":{"domain":"","port":0},"bstation_net":"","system_net":{"dns":"192.168.1.1","dns2":"0.0.0.0","subnet":"255.255.255.0","gateway":"192.168.1.1"},"link_vol_text":"Linked Audio","file_indicator":1730548800},"chk_obj":1067956892},"vars":{"obj":{"items":[]},"chk_obj":1067956892},"con_devs":[{"obj":{"fav":{}},"id":2,"type ":0}],"rooms":[{"id":1,"info":{"obj":{"name":"Living Room","room_image":"R:1.JPG","background":"R:0.JPG","status_bar":[{"idx":0,"type":0,"model_id":-1},{"idx":1,"type":0,"model_id":-1},{"idx":2,"type":0,"model_id":-1},{"idx":3,"type":0,"model_id":-1}],"startup_dev":20482,"main_dev":20482,"lights_dev":-1,"fav_dev":-1},"chk_obj":1067956892},"menu_devs":{"obj":[{"id":20482,"name":"Main","link_btn_id":-1,"hard_btns":[{"id":-1,"hbtn_id":4625,"macro":[{"type":4,"j_type":0,"target":7}]},{"id":-1,"hbtn_id":4624,"macro":[{"type":54,"room":1}]}],"menu_buttons":[{"id":2049,"name":"Living Room TV","image":"1.png","macro":[{"type":33,"code":67585}]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]}]}],"chk_obj":1067956892},"devices":[{"obj":{"id":2,"name":"Living Room TV","model_name":"Living Room TV","type":0,"link_btn_id":2049,"device_type":"DBMODEL","hard_btns":[{"id":-1,"hbtn_id":275,"macro":[{"type":33,"code":131347}]},{"id":-1,"hbtn_id":276,"macro":[{"type":33,"code":131348}]},{"id":-1,"hbtn_id":277,"macro":[{"type":33,"code":131349}]},{"id":-1,"hbtn_id":4625,"macro":[{"type":4,"j_type":0,"target":7}]},{"id":-1,"hbtn_id":4624,"macro":[{"type":54,"room":1}]}],"pages":[{"type":4,"name":"channel","id":0,"buttons":[{"id":279,"btn_type":0,"hbtn_id":279,"name":"CH +","idx":0,"ui_id":"279","macro":[{"type":33,"code":131351}]},{"id":280,"btn_type":0,"hbtn_id":280,"name":"CH -","idx":1,"ui_id":"280","macro":[{"type":33,"code":131352}]},{"id":-1,"btn_type":0,"hbtn_id":278,"name":"","idx":2,"hidden":2,"hidden_text":1,"ui_id":"278","macro":[]}]}],"quick_bar":[{"id":0},{"id":1},{"id":2},{"id":3},{"id":4},{"id":5},{"id":6},{"id":7},{"id":8},{"id":9}],"keyboard":{"con_dev_id":2,"keyboard_id":0}},"chk_obj":1067956892}],"scenes":{"obj":[],"chk_obj":1067956892}}],"camera_popups":{"obj":[],"chk_obj":1067956892},"trfzws":{"obj":[],"chk_obj":1067956892},"notifications":{"obj":[],"chk_obj":1067956892},"door_stations":{"obj":[],"chk_obj":1067956892}},"header":{"version":65536,"chksum":0,"type":"Common","firmware":0}}
```
*(Note: `131347 = (2 << 16) | 275`, `131348 = (2 << 16) | 276`, and `131349 = (2 << 16) | 277` are Vol+, Vol-, and Mute on room device object 2. That object controls connected device 2 (`keyboard.con_dev_id`); the equal numbers are a choice made for this vector. `67585 = (1 << 16) | 2049` assumes that room 1's menu object id is 1. Hard button 4625 jumps to the room main menu, and 4624 turns room 1 off.)*
---

## 9. Open Items

1. **Non-empty `builtins` schema:** Identify real-world device drivers that populate non-empty `con_devs[].db.builtins` arrays (observed empty across all devices in the reference project).
2. **Multi-station master/slave topology:** Document `bstations` array cardinality and network routing rules when multiple MRX controllers operate in a distributed multi-controller system.
