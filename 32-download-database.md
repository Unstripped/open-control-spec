---
title: Controller Binary Download Database (DnData.dn)
version: "0.5.1"
status: draft
---

# Controller Binary Download Database (`DnData.dn`)

## 1. Overview & Role

The controller binary download database (`/remote/Store/data/DnData.dn`) is the compiled binary runtime database consumed by the master system automation daemon (`BS.exe`) on URC Total Control MRX processors. While `mcs.dat` and `mcs_info.dat` provide high-level JSON topology to client remotes and auxiliary daemons (`rmsmcsman.exe`, `rui_svrd.exe`), `DnData.dn` drives low-level hardware control, IR/serial command execution, zone macro execution, client activation data, and protected configuration blocks.

At system startup, the automation engine (`BS.exe`) opens `DnData.dn` in read-only mode and loads all configuration tables into memory.

---

## 2. Container Framing & Primary Header

`DnData.dn` begins with a 56-byte primary header, followed by project identification metadata and a directory of 26 table offsets.

### 2.1 Primary Header (0x00 - 0x38, 56 bytes)

| Offset | Size | Type | Value / Meaning | Confidence |
|---|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | First-table offset minus 4 (e.g. `0x000000AF` when first table is at `0x000000B3`) | inferred |
| `+0x04` | 8 | `uint8[8]` | Project generation identifier / timestamp hash (any 8 bytes accepted) | static |
| `+0x0C` | 32 | `string` | Target hardware model name (null-padded ASCII, e.g. `MRX-15\0`) | confirmed |
| `+0x2C` | 4 | `uint32_le` | Total file size in bytes | inferred |
| `+0x30` | 4 | `uint32_le` | Database format version (`0x00117000` = v1.17.0 Master DB) | confirmed |
| `+0x34` | 4 | `uint32_le` | Database role (`0` = Master Controller DB, `0xFFFF` = Slave DB) | confirmed |
### 2.2 Project Metadata (Variable length, begins at 0x38)

Immediately following the 56-byte primary header:

| Offset | Size | Type | Value / Meaning | Confidence |
|---|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | Model ID (`1` observed) | confirmed |
| `+0x04` | 2 | `uint16_le` | Project Name length $N$ (including null terminator, e.g. `15`) | confirmed |
| `+0x06` | $N$ | `string` | Project Name (null-terminated ASCII, e.g. `Sample Project\0`) | confirmed |

### 2.3 Table Offset Directory (104 bytes)

Immediately following the project name, an array of 26 32-bit little-endian integers specifies the byte offset of each database table from the start of the file:

```
offset[0..25] = uint32_le * 26
```

Table payloads follow consecutively in the file. All table offsets are absolute file offsets.

---

## 3. Table Catalog & Specifications

The database contains 26 recognized table indices:

| Index | Offset (Ref) | Name | Purpose / Contents | Ref Item Count |
|---|---|---|---|---|
| 0 | 179 | configuration table | System configuration, location, sunrise/sunset, and the protected configuration block (§4.1) | 1 |
| 1 | 3,362 | access-point table | Wi-Fi access point profiles | 0 |
| 2 | 3,370 | socket table | Internal socket connection definitions | 12 |
| 3 | 3,450 | base-station table | Base station controllers in topology (master/slaves) | 1 |
| 4 | 3,529 | client table | User interface clients (touchscreens, remotes, apps) | 3 |
| 5 | 3,948 | connected-device table | Controlled AV and automation hardware devices | 8 |
| 6 | 94,531 | stream-routing table | DMS digital audio streaming source routing | 7,169 |
| 7 | 94,587 | system-info flag | System info flag / state byte | 0 |
| 8 | 94,588 | integer-variable table | User integer variables | 0 |
| 9 | 94,596 | boolean-variable table | User boolean variables | 0 |
| 10 | 94,604 | room table | Physical room/zone definitions and macro bindings | 3 |
| 11 | 114,134 | sensor-trigger table | Sensor threshold trigger rules | 0 |
| 12 | 114,142 | trigger-code table | Macro trigger event code mapping table | 234 |
| 13 | 115,554 | system-macro table | Global system automation macros | 0 |
| 14 | 115,562 | system-off macro | System Off macro sequence steps | 4 |
| 15 | 115,686 | event-timer table | Scheduled event timer rules | 4 |
| 16 | 115,708 | global power-macro table | Global power management macro bindings | 0 |
| 17 | 115,716 | lighting-scene table | Lighting scene controller definitions | 4 |
| 18 | 115,736 | lighting-gateway table | Advanced lighting gateway objects | 1 |
| 19 | 115,807 | camera-popup table | Door station / camera video popup triggers | 0 |
| 20 | 115,815 | weekly-schedule table | Weekly scheduled calendar events | 4 |
| 21 | 115,867 | secondary-controller power table | Secondary controller power bus controls | 0 |
| 22 | 115,875 | notification table | Push notification action bindings | 0 |
| 23 | 115,883 | email-recipient table | User email notification addresses | 1 |
| 24 | 115,890 | door-station table | Intercom door station profiles | 0 |
| 25 | 115,898 | Z-Wave gateway table | TRF-ZW10 Z-Wave gateway descriptor | 0 |

---

## 4. Cryptographic Envelopes

Protected payload blocks embedded within table records follow a standard 8-byte framing prefix:

| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | `plain_size` ($S_{org}$) |
| `+0x04` | 4 | `uint32_le` | `cipher_size` ($S_{enc}$, 16-byte aligned) |
| `+0x08` | $S_{enc}$ | `uint8[]` | AES-128 ECB ciphertext |

### 4.1 Configuration Table Protected Block (Table 0)
- **Location:** Table 0, offset `+0x0F` (file offset 194 in the reference project).
- **Framing:** 8-byte prefix (§4) followed by AES-128 ECB ciphertext.
- **Key:** Encrypted with a fixed 16-byte content key held by the controller. This specification does not provide that key or describe how to obtain it.
- **Contents:** The block carries the project security key used for Two-Way module scripts (see [Containers](21-containers.md) §4) and the project's network credentials. Its plaintext layout is not specified.
- **Keyless form:** A programmer that does not produce this block MUST emit the empty Table 0 form (§4.3). Without a project key the controller cannot decrypt Two-Way module scripts `[INFERRED]`.

### 4.2 Client and Device Envelopes (Table 4, Table 5)
- Client-table records carry a 16-byte client activation descriptor encrypted with the controller content key. Its contents are not specified, and this specification does not describe how to create one.
- Connected-device table records carry serial and network endpoint parameters (IP address, network port, serial baud) in envelopes encrypted with the controller content key (§5.3). This specification does not provide that key.

---
### 4.3 Empty Table Representations

Tables without active records serialize in one of two standardized forms:
1. **Table 0 (configuration table):** Serializes as a 5-byte sequence `00 00 00 00 00` (4 zero bytes representing size 0, followed by 1 byte alignment padding). The controller treats a zero first word as an empty table `[CONFIRMED]`.
2. **All Other Tables (Tables 1..4, 6..9, 11, 13, 15..25):** Serialize as an 8-byte empty header: `04 00 00 00 00 00 00 00` (4-byte table size = 4, followed by 4-byte item count = 0) `[CONFIRMED]`.

---

### 5.0 Table 8: Integer-Variable Table

Table 8 contains user-defined runtime integer and boolean variables accessible via macro operations and runtime query opcode `0x089A`:

#### 5.0.1 Table Header
| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | Table byte length (excluding this 4-byte field) |
| `+0x04` | 4 | `uint32_le` | Variable record count $N_v$ |

#### 5.0.2 Variable Record
Each variable record consists of an 8-byte record header followed by variable parameters:

| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | Variable ID (`var_id`) |
| `+0x04` | 4 | `uint32_le` | Record payload size $S_{rec} = 13 + L_{val} + L_{name}$ |
| `+0x08` | 1 | `uint8` | Variable type (`0` = Boolean, `1` = Integer, `2` = String) |
| `+0x09` | 1 | `uint8` | Initial value byte length $L_{val}$ (e.g. 1 for boolean/byte, 4 for int32) |
| `+0x0A` | $L_{val}$ | `bytes` | Initial value bytes (little-endian) |
| var | 1 | `uint8` | String display formatting type (`string_format`) |
| var | 4 | `int32_le` | Minimum allowed value (`min_val`) |
| var | 4 | `int32_le` | Maximum allowed value (`max_val`) |
| var | 2 | `uint16_le` | Variable name byte length $L_{name}$ |
| var | $L_{name}$ | `string` | Variable name string (ASCII) |


## 5. Table Framing & Structural Semantics

### 5.1 Standard List Table Header
Most list tables (such as Tables 1, 2, 3, 4, 5, 8, 9, 10, 11, 12, 13, 16, 19, 21, 22, 24) share a common 8-byte header:

| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | Table byte length (excluding this 4-byte field) |
| `+0x04` | 4 | `uint32_le` | Record count $N$ |

If record count $N = 0$, the table has size `4` and contains no records.

### 5.2 Table 4: Client Table Records
Each client record starts with an 8-byte entry header followed by client identity, model descriptors, sub-type flags, activation envelopes, and trailer structures:

| Offset | Size | Type | Description | Confidence |
|---|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | Client ID (corresponds 1:1 with `clients[].obj_id` in `mcs_info.dat`) | confirmed |
| `+0x04` | 4 | `uint32_le` | Total client payload size $S_{client}$ (excluding this 8-byte header) | confirmed |
| `+0x08` | 2 | `uint16_le` | Client display name length $L_{name}$ | confirmed |
| `+0x0A` | $L_{name}$ | `string` | Client display name (null-terminated ASCII, e.g. `iPhone Mobile\0`) | confirmed |
| $+0$ | 1 | `uint8` | Model string length $L_{model}$ | confirmed |
| $+1$ | $L_{model}$ | `string` | Hardware model string (e.g. `IPhoneV2\0` or `IPadV2\0`) | confirmed |
| $+0$ | 4 | `uint32_le` | Client Type (`1` = Mobile Application iOS/Android, `0` = Dedicated Remote/Touchscreen) | confirmed |
| $+4$ | 4 | `uint32_le` | Client Sub-Type (`0x0F02` = Mobile Client Profile) | confirmed |
| $+8$ | 4 | `uint32_le` | Activation descriptor length indicator (`0x00001000` = 16 bytes) | confirmed |
| $+12$ | 16 | `uint8[16]` | Client activation descriptor (encrypted with the controller content key; contents not specified) | confirmed |
| $+28$ | 4 | `uint32_le` | Secondary block indicator (`0x00000101`) | confirmed |
| $+32$ | 4 | `uint32_le` | Secondary block length (`0x00000010` = 16 bytes) | confirmed |
| $+36$ | 16 | `uint8[16]` | Secondary opaque 16-byte token (contents not specified) | confirmed |
| $+52$ | 4 | `uint32_le` | Tertiary block indicator (`0x00000100`) | confirmed |
| $+56$ | 4 | `uint32_le` | Tertiary block length (`0x00000010` = 16 bytes) | confirmed |
| $+60$ | 16 | `uint8[16]` | Tertiary opaque 16-byte token (contents not specified) | confirmed |
| $+76$ | 5 | `uint8[5]` | Reserved flags (`\xFF\xFF\xFF\xFF\x00` observed) | confirmed |
| $+81$ | 8 | `uint8[8]` | Button action block header (`\x04\x00\x00\x00\x00\x00\x00\x00` empty count) | confirmed |
| $+89$ | 8 | `uint8[8]` | Auxiliary action block header (`\x04\x00\x00\x00\x00\x00\x00\x00` empty count) | confirmed |
| $+97$ | 3 | `uint8[3]` | Reserved tail alignment bytes (`\x00\x00\x00`) | confirmed |

### 5.3 Table 5: Connected-Device Table Records
Each connected device record starts with an 8-byte entry header followed by device metadata, command sets, and transport definitions:

| Offset | Size | Type | Description | Confidence |
|---|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | Device ID (corresponds 1:1 with `con_devs[].id` in `mcs.dat` and `devices[].obj_id` in `mcs_info.dat`) | confirmed |
| `+0x04` | 4 | `uint32_le` | Total device object payload size $S_{obj}$ (excluding this 8-byte header) | confirmed |
| `+0x08` | 2 | `uint16_le` | Device name length $L_{name}$ | confirmed |
| `+0x0A` | $L_{name}$ | `string` | Device name (null-terminated ASCII, e.g. `APPLE TV Great Room\0`) | confirmed |
| $+0$ | 1 | `uint8` | Device model type length $L_{type}$ | confirmed |
| $+1$ | $L_{type}$ | `string` | Device model type (e.g. `DBMODEL\0` or `HDA-8100\0`) | confirmed |
| $+0$ | 4 | `uint32_le` | Assigned Room ID (`0` if unassigned) | confirmed |
| $+4$ | 4 | `uint32_le` | Sensor Base Station ID (`0xFFFFFFFF` if unassigned) | confirmed |
| $+8$ | 1 | `uint8` | Sensor Port index | confirmed |
| $+9$ | 1 | `uint8` | Sensor Type | confirmed |
| $+10$ | 1 | `uint8` | Line of Sight flag (`0` = standard controller control, `1` = line of sight only) | confirmed |
| $+11$ | 1 | `uint8` | Power Manager required flag (`0` = no, `1` = yes) | confirmed |
| $+12$ | 1 | `uint8` | Skip Power Track flag (`0` = track, `1` = skip) | confirmed |
| $+13$ | 4 | `uint32_le` | Favorite object table size $S_{fav}$ | confirmed |
| $+17$ | $S_{fav}$ | `record` | Favorite object table payload | confirmed |
| $+0$ | 5 | `uint8[5]` | Favorite usage block. Byte 4 is `favorite_flag` (`0` = unused, `1` = used) | confirmed |
| $+5$ | 7 | `uint8[7]` | Favorite timing parameters (present only if `favorite_flag == 1`) | confirmed |
| $+0$ | 2 | `uint16_le` | Device sub-type (`0x0105` = Multi-zone amplifier HDA, `0x0770` = Apple TV, `0x0790` = Set-top box, `0x0710` = Streaming audio, `0x0720` = TV) | confirmed |
| $+2$ | variable | `record` | Subdevice architecture (HDA zone splits and outputs vs standard input/zone tables) | confirmed |
| $+0$ | 4 | `uint32_le` | Event file name length $L_{evt}$ | confirmed |
| $+4$ | $L_{evt}$ | `string` | Event file name (null-terminated ASCII) | confirmed |
| $+0$ | 1 | `uint8` | Target command transport type: `0` = IR, `1` = IR Blaster, `2` = Serial, `3` = Network, `7` = Zigbee | confirmed |
| $+1$ | variable | `record` | Target transport parameters: Base station ID (uint32) + port (uint8) for IR; 8-byte AES-128 ECB envelope + timeout/mode for Serial/Network | confirmed |
| $+0$ | 1 | `uint8` | 2-Way target flag (`0` = none, `1` = IP 2-Way, `2` = Serial 2-Way) | confirmed |
| $+1$ | variable | `record` | 2-Way target parameters: 8-byte AES-128 envelope with IP/port when `is_2way == 1` | confirmed |
| $+0$ | 1 | `uint8` | Flag byte (`0x00` observed) | confirmed |
| $+1$ | 1 | `uint8` | DB structure type (`0` or `1` = single/multi-zone DB with built-in commands; `2` = multi-zone DB tree) | confirmed |
| $+2$ | variable | `record` | Database command tables & zone bindings (see §5.2.1) | confirmed |
| $+0$ | 4 | `uint32_le` | Master script length $L_{script}$ | confirmed |
| $+4$ | $L_{script}$ | `string` | Master script name (null-terminated ASCII) | confirmed |
| $+0$ | 4 | `uint32_le` | System parameters original size $S_{org}$ | confirmed |
| $+4$ | 4 | `uint32_le` | System parameters encrypted size $S_{enc}$ | confirmed |
| $+8$ | $S_{enc}$ | `uint8[]` | AES-128 ECB encrypted system parameters (absent if $S_{org} == 0$) | confirmed |
| $+0$ | 4 | `uint32_le` | Unique instance ID (`0xFFFFFFFF` if unassigned) | confirmed |
| $+4$ | 1 | `uint8` | Feedback type (`0` = none, `1` = IP Device, `2` = Serial via Base Station) | confirmed |
| $+5$ | variable | `record` | Feedback configuration: 8-byte AES envelope, zone output feedback count $N_{zo}$, and $N_{zo}$ zone feedback option blocks | confirmed |
| $+0$ | 4 | `uint32_le` | Universal config file name length $L_{univ}$ | confirmed |
| $+4$ | $L_{univ}$ | `string` | Universal config file name | confirmed |
| $+0$ | 1 | `uint8` | Power-on delay for room link (seconds) | confirmed |
| $+1$ | 1 | `uint8` | Special object type (`0` = standard device, `2` = special ring device) | confirmed |

#### 5.2.1 Device Command Blocks & Zone Bindings
Under DB structure types `0` and `1`:
1. **Built-in DB Count ($N_{db}$):** 4 bytes `uint32_le`. For each database:
   - DB name length (uint16_le) and name string.
   - Command block size $S_{cmd}$ (uint32_le). If non-zero, begins with 2 flag bytes, command count $N_c$ (uint32_le), and $N_c$ command records.
   - Has 2-way module flag (uint8). If non-zero, module name length (uint32_le) and name string.
2. **Zone Count ($N_{zone}$):** 4 bytes `uint32_le`. For each zone:
   - Has zone info flag (uint8). If non-zero, 40-byte block: zone ID (uint32), Power On ID (uint32), Power Off ID (uint32), Power Toggle ID (uint32), Mute On ID (uint32), Mute Off ID (uint32), Mute Toggle ID (uint32), Volume Up ID (uint32), Volume Down ID (uint32), and input ID count $N_{inp}$ (uint32).
   - $N_{inp} \times 4$ bytes of input IDs.
   - Built-in input count $N_{binp}$ (uint32_le) followed by $N_{binp} \times 4$ bytes.
   - Zone command block ($S_{zcmd}$ uint32_le followed by command records).
   - Zone 2-way module flag and string.
   - Zone keyboard command block ($S_{kb}$ uint32_le followed by keyboard command records).

**Command record.** Every command record in a built-in, zone, or keyboard command block has this layout:

| Offset | Size | Type | Field | Confidence |
|---|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | Command ID (device function ID, e.g. `275` volume up) | confirmed |
| `+0x04` | 1 | `uint8` | Repeat option. Observed `0` on volume commands and `0` or `1` on mute commands | static (name) |
| `+0x05` | 4 | `uint32_le` | Minimum press-hold count. Observed `1` | static (name) |
| `+0x09` | 4 | `uint32_le` | Repeat count. Observed `1`–`3` | static (name) |
| `+0x0D` | 4 | `uint32_le` | Ramp start. Observed `100` and `300`; unit not established | static (name) |
| `+0x11` | 4 | `uint32_le` | Ramp speed. Observed `100` | static (name) |
| `+0x15` | 2 | `uint16_le` | Command body length $L$ | confirmed |
| `+0x17` | $L$ | `bytes` | Command body (IR code, serial string, or IP payload such as a JSON template) | confirmed |
| `+0x17+L` | 1 | `uint8` | Is-macro flag. If non-zero, a `uint32_le` step count $N$ and $N \times 8$ bytes of steps follow | confirmed |

The controller reads the repeat and ramp fields when it looks up a command to send (static). During a held press it sends the command once immediately, then repeats it after the ramp-start interval and at ramp-speed intervals while the button is held; the hold ends on any client byte or after a fixed 1000 ms (`52-control-protocol` §3.1.4). In the reference, the `OP_DEV_CMD` macro step (§5.4.2) repeats the target command's repeat count, ramp start, and ramp speed as `uint32_le` values at offsets `+27`, `+31`, and `+35`, counted from the step's `0x21` opcode byte (confirmed by comparison across four devices).

---

### 5.3 Table 10: Room Table Records
Each room record starts with an 8-byte entry header followed by physical room metadata, zone routing, menu buttons, inner device objects, and automation rules:

| Offset | Size | Type | Description | Confidence |
|---|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | Room ID (corresponds 1:1 with `zones[].id` in `mcs.dat` and `room_ids` in `Link.dat`, `Event.dat`, `Alarm.dat`) | confirmed |
| `+0x04` | 4 | `uint32_le` | Total room object payload size $S_{room}$ | confirmed |
| `+0x08` | 4 | `uint32_le` | Real Room ID | confirmed |
| `+0x0C` | 2 | `uint16_le` | Room name length $L_{name}$ | confirmed |
| `+0x0E` | $L_{name}$ | `string` | Room name (null-terminated ASCII, e.g. `Living Room\0`) | confirmed |
| $+0$ | 1 | `uint8` | Room is hidden flag (`0` = visible on UI clients, `1` = hidden / utility zone) | confirmed |
| $+1$ | 1 | `uint8` | Room flag byte (`0x00` observed) | confirmed |
| $+2$ | 12 | `uint8[12]` | AVR / DMS zone routing descriptor: enable link (uint8), DMS/AVR ID (uint32), zone index (uint8), AVR input index (uint8), DMS condev ID (uint32), DMS output index (uint8). The zone index is the amplifier's zone-output index (the same 0-based index used by Table 5 zone-output lists and `OP_DEV_CMD` zone bindings); a room-level volume or mute request is routed to that zone of the linked device `[STATIC]`. In the reference installation the room linked to the 24-zone amplifier stores `01 02000000 00 00 FFFFFFFF 00`: link enabled, DMS/AVR ID = connected device `2`, zone index `0` | confirmed (layout) / static (zone index) |
| $+14$ | variable | `record` | Music activity table: 4-byte size, 4-byte count, 11-byte records | confirmed |
| $+0$ | variable | `record` | Connected device ID table: 4-byte size, 4-byte count $N_{dev}$, followed by $N_{dev} \times 4$ bytes of assigned device IDs | confirmed |
| $+0$ | variable | `record` | Room menu (see §5.3.1) | confirmed |
| $+0$ | variable | `record` | Device-object table (see §5.3.2) | confirmed |
| $+0$ | variable | `record` | Module-device table: 4-byte size, 4-byte count, 50-byte records | confirmed |
| $+0$ | variable | `record` | Room power-on button record: 4-byte action count followed by macro actions | confirmed |
| $+0$ | variable | `record` | Room Power Off button record: 4-byte action count followed by macro actions | confirmed |
| $+0$ | variable | `record` | Room Command Table: 4-byte size, 4-byte count, 8-byte command mappings | confirmed |
| $+0$ | variable | `record` | Room Alarm Table: 4-byte size, 4-byte count, alarm schedule descriptors | confirmed |
| $+0$ | variable | `record` | Room Sleep Timer Table: 4-byte size, 4-byte count, sleep timer rules | confirmed |
| $+0$ | variable | `record` | Room Power Table: 4-byte size, 4-byte count | confirmed |
| $+0$ | variable | `record` | Room Device Power Table: 4-byte size, 4-byte count | confirmed |
| $+0$ | variable | `record` | Room Scene List: 4-byte size, flag (uint8), scene table ID (uint32), scene count, 2-byte scene IDs | confirmed |
| $+0$ | variable | `record` | Lighting-gateway room object: 4-byte size, scene ID table, light item table | confirmed |
| $+0$ | 1 | `uint8` | TC15 Room Scene Object flag (`0` = absent, `1` = present) | confirmed |
| $+1$ | variable | `record` | TC15 Room Scene Object Table: 4-byte size, 4-byte count, 49-byte records | confirmed |

#### 5.3.1 Room Menu
Defines zone user interface buttons and menu navigation bindings:
- **Header:** Total menu size (uint32_le), Menu ID (uint32_le), Client menu count $N_{cm}$ (uint32_le), and $N_{cm} \times 4$ bytes client menu IDs. The Menu ID is a project-wide UI object id. It is the high 16 bits of the `code` on the room's `menu_devs` buttons in `core_model.json`; in the reference these are menu IDs `1`, `3`, and `5` for rooms 0–2. The client menu IDs are `20481 + room_id`, the `menu_devs.obj[].id` values `[CONFIRMED]`.
- **Punch Through Table:** Has punch flag (uint32_le). If non-zero (`6` observed), followed by 42 bytes: six 7-byte bindings in the order power, volume, channel, navigation, transport, numeric keypad. Each binding has the same layout as the device punch records (§5.3.2). In the reference, only the volume binding is set. It names the room's audio device: connected device `12` for one room, and connected device `2` (the 24-zone amplifier) with zone index `0` for another `[CONFIRMED]`.
- **Button Count ($N_b$):** 4 bytes `uint32_le`. Followed by $N_b$ button records. Each button has:
  - Button ID (uint32_le).
  - Button size (uint32_le).
  - Action count (uint32_le).
  - Macro action sequence (see §5.4).

#### 5.3.2 Device-Object Table
Defines the logical UI devices bound into this room:
- **Header:** Table size (uint32_le) and device count $N_d$ (uint32_le).
- **Device Objects:** For each device:
  - Device ID (`uint32_le`) and object size (`uint32_le`). The Device ID is a project-wide UI object id. It equals `rooms[].devices[].obj.id` in `core_model.json`, **not** the connected-device id `[CONFIRMED]`.
  - Name length (`uint16_le`) and null-terminated name string.
  - **Device Configuration Header (9 bytes):**
    | Offset | Size | Type | Description |
    |---|---|---|---|
    | `+0x00` | 1 | `uint8` | Unknown; `0` or `1` observed |
    | `+0x01` | 1 | `uint8` | Unknown; `0` observed |
    | `+0x02` | 4 | `uint32_le` | Target connected device ID (`mcs.dat` `con_devs[].id`) `[CONFIRMED]`. Equal to `obj.keyboard.con_dev_id` for all 7 reference device objects |
    | `+0x06` | 3 | `uint8[3]` | Unknown; `01 00 00` observed |
  - **2-Way Module Info Block:** 4-byte size $S_{mod}$ followed by $S_{mod}$ bytes module configuration payload.
  - **Punch Configuration Block (47 bytes):**
    | Offset | Size | Type | Description |
    |---|---|---|---|
    | `+0x00` | 4 | `uint32_le` | Punch flag (`6` observed) |
    | `+0x04` | 7 | `record` | Power punch: `con_dev_id` (uint32), `db_type` (uint8), `zone_index` (uint8), `db_index` (uint8) |
    | `+0x0B` | 7 | `record` | Volume punch: `con_dev_id` (uint32), `db_type` (uint8), `zone_index` (uint8), `db_index` (uint8) |
    | `+0x12` | 7 | `record` | Channel punch: `con_dev_id` (uint32), `db_type` (uint8), `zone_index` (uint8), `db_index` (uint8) |
    | `+0x19` | 7 | `record` | Navigation punch: `con_dev_id` (uint32), `db_type` (uint8), `zone_index` (uint8), `db_index` (uint8) |
    | `+0x20` | 7 | `record` | Playback transport punch: `con_dev_id` (uint32), `db_type` (uint8), `zone_index` (uint8), `db_index` (uint8) |
    | `+0x27` | 7 | `record` | Numeric keypad punch: `con_dev_id` (uint32), `db_type` (uint8), `zone_index` (uint8), `db_index` (uint8) |
    | `+0x2E` | 1 | `uint8` | Unknown; `0` observed |

    An unset binding is `FF FF FF FF 00 00 00`. Offsets are `[CONFIRMED]` by parsing all 7 reference device objects. Six of them have a volume punch of `0C 00 00 00 01 00 00` (connected device 12, zone 0) or `02 00 00 00 01 00 00` (24-zone amplifier, zone 0). The seventh, device object 12, binds its own connected device 10 (`0A 00 00 00 01 00 00`). Device objects whose button list lacks `275`–`277` take volume through this binding `[INFERRED]` (not live-tested).
  - Button count (`uint32_le`) and an array of button records containing the device's macro action sequences.

#### 5.3.3 Module-Device Table
Defines Two-Way integration module bindings to room zones:
- **Header:** Table size (`uint32_le`) and module device count $N_m$ (`uint32_le`).
- **Module Device Record (50 bytes each):**
  | Offset | Size | Type | Description |
  |---|---|---|---|
  | `+0x00` | 4 | `uint32_le` | Module Device ID (`mod_dev_id`) |
  | `+0x04` | 4 | `uint32_le` | Zone binding count (`6` observed) |
  | `+0x08` | 42 | `record[6]` | 6 zone/db binding tuples (7 bytes each: `con_dev_id: uint32_le, db_type: uint8, zone_index: uint8, db_index: uint8`) |

---

### 5.4 Macro Action Sequences
Automation macros and UI button actions are encoded as sequences of variable-length action records.

#### 5.4.1 Action Record Framing
| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | Action record payload size $S_{act}$ (excluding this 4-byte field) |
| `+0x04` | 1 | `uint8` | Action group type (`1` = automation/control action) |
| `+0x05` | 1 | `uint8` | Action opcode $O$ (first byte of action payload) |
| `+0x06` | $S_{act} - 2$ | `uint8[]` | Action parameters |

#### 5.4.2 Recognized Action Opcodes
| Opcode | Hex | Size (Typ) | Meaning / Parameters |
|---|---|---|---|
| `OP_DELAY` | `0x01` | 7 | Delay step: duration in milliseconds (`uint32_le`), flags (`uint16_le`). Examples: `0x00001388` = 5000 ms, `0x00000064` = 100 ms. |
| `OP_JUMP` | `0x04` | 17 / 24 | UI navigation step. Parameters: jump type (`uint8`: `0` default, `1` page, `2` device, `3` sub-menu, `4` special screen), option (`uint8`: the default-jump destination for type 0, or the special-screen code for type 4), target object ID (`uint32_le`, `0xFFFFFFFF` = none), page ID (`uint32_le`, `0xFFFFFFFF` = none), transition-style string length (`uint32_le`, counting the NUL), and the style string (`"Replace"` or empty). Examples: `04 02 00 07000000 FFFFFFFF 08000000 "Replace\0"` jumps to device object 7. `04 04 0B FFFFFFFF FFFFFFFF …` opens special screen 11 (off-site). `04 04 12 …` opens special screen 18 (sleep timer). `04 04 60 …` opens special screen 96 (room EQ) `[CONFIRMED]` (layout) / `[STATIC]` (enumerations). The controller does not execute a jump itself; it forwards the jump to the client that triggered the macro `[STATIC]`. `core_model.json` step type `4` uses the same enumeration (`53-client-protocols` §3.4.4). |
| `OP_DEV_CMD` | `0x21` | 44 | Device command step: target device ID (`uint32_le`), target command ID (`uint32_le`), repeat count, delay interval. |
| `OP_ZONE_CMD` | `0x35` | 11 | Zone audio control step: target zone ID, volume level or mute toggle. |
| `OP_ROOM_OFF` | `0x36` | 11 | Room power-off step: sub-type (`uint8`, `0x00` observed), room ID (`uint32_le`), then `uint32_le` `0xFFFFFFFF` (meaning unresolved). Example: `36 00 01000000 FFFFFFFF` turns room 1 off `[CONFIRMED]` (layout). The same opcode is `core_model.json` step type `54` (`{"type": 54, "room": <id>}`) `[STATIC]`. |
| `OP_VAR_STATE` | `0x37` | 11 | Variable / room power state mutation: sub-op `0x00` (`uint8`), variable ID (`uint32_le`), new value (`uint32_le`). Wire payload is 10 bytes (`37 00 <var_id u32> <val u32>`). |
| `OP_JSON_BROADCAST`| `0x39` | Variable | JSON broadcast command step: flags, string length (`uint32_le`), JSON string payload (e.g. `{"type":"set","zone":{"id":"all","ctrl":{"power":"off"}}}`). |

**Default-jump destinations (`OP_JUMP` type 0).** The option byte selects: `0` = return to the current/last device, `1`/`2` = previous/next page, `3`/`4` = previous/next navigation page, `5`/`6` = previous/next device, `7` = room main menu, `8` = room list `[STATIC]`. Values above `8` are undefined; the controller does not interpret them, and a client MUST treat them as a no-op and continue with the next step `[STATIC]`. The reference database contains no type-0 `OP_JUMP` records `[CONFIRMED]`. Type-4 special-screen codes are a separate value space (see `53-client-protocols` §3.4.4).

---

### 5.5 Table 12: Trigger-Code Table
Maps 16-bit incoming automation trigger codes (from schedules, client remotes, or external sensor thresholds) to 32-bit execution macro IDs:

| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | Table byte length ($N \times 6 + 4$) |
| `+0x04` | 4 | `uint32_le` | Record count $N$ (`234` in reference installation) |
| `+0x08` | $N \times 6$ | `record[]` | Array of 6-byte trigger code entries |

Each 6-byte record consists of:
- `code`: `uint16_le` (e.g. `8193` / `0x2001`)
- `macro_id`: `uint32_le` (e.g. `459031` / `0x00070117`)

---

### 5.6 Table 14: System-Off Macro
Encodes the master System Off macro sequence executed when the global power-off button or system sleep timer triggers:

| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | Total button object size $S_{btn}$ |
| `+0x04` | 4 | `uint32_le` | Action step count $N_{act}$ (`4` in reference installation) |
| `+0x08` | variable | action record[] | Ordered macro actions |

In the reference installation, Table 14 contains 4 ordered action steps:
1. **Step 0 (`0x39` JSON Broadcast, 67 bytes):** Broadcasts zone power-off JSON: `{"type":"set","zone":{"id":"all","ctrl":{"power":"off"}}}`.
2. **Step 1 (`0x36` Room Trigger, 11 bytes):** Triggers room 0 power-off macro (`room_id = 0`).
3. **Step 2 (`0x36` Room Trigger, 11 bytes):** Triggers room 1 power-off macro (`room_id = 1`).
4. **Step 3 (`0x36` Room Trigger, 11 bytes):** Triggers room 2 power-off macro (`room_id = 2`).

---

## 6. Topology Cross-References

`DnData.dn` entities link directly to the JSON topology files and automation tables:

```
mcs_info.dat: bstations[].id <====> DnData.dn Table 3 (base-station table)
mcs_info.dat: clients[].id   <====> DnData.dn Table 4 (client table)
mcs.dat:      con_devs[].id  <====> DnData.dn Table 5 (connected-device table)
mcs_info.dat: devices[].obj_id<===> DnData.dn Table 5 (connected-device table)
core_model.json: rooms[].devices[].obj.id <====> DnData.dn Table 10 device-object table device ID
core_model.json: rooms[].devices[].obj.keyboard.con_dev_id <====> device-object table header con_dev_id (reference)
mcs.dat:      zones[].id     <====> DnData.dn Table 10 (room table)
Link.dat:     room_ids[]     <====> DnData.dn Table 10 (room table)
Event.dat:    room_id        <====> DnData.dn Table 10 (room table)
Alarm.dat:    room_id        <====> DnData.dn Table 10 (room table)
Event.dat:    event_macro_id <====> DnData.dn Table 12 (trigger-code table)
```

---

## 7. Verification & Round-Trip Validation

Top-level database framing, directory structures, and inner record models can be validated via round-trip deserialization and byte-exact serialization against reference controller databases. An implementation SHOULD verify byte-exact round-trip of the top-level framing and of the connected-device (Table 5) and room (Table 10) inner records against sample databases.

---

## 8. Open Items

1. **Exhaustive Parameter Permutations for Rare Transports:** While IR, Serial, and IP target devices are fully decoded and byte-exact, uncommon transport modes like Zigbee direct binding (`target_type == 7`) have fixed 8-byte framing documented from limited observation, without a live Zigbee gateway fixture.
