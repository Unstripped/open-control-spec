---
title: "Mobile and Remote Client Protocols"
section: "53-client-protocols"
version: "0.2.0"
status: review
circumvention: false
---

# Mobile and Remote Client Protocols

## 1. Scope

This section specifies the interface contracts, data models, and network exchanges required for mobile applications (iOS, iPadOS, Android) and hardware remotes/keypads to discover an MRX controller, load project configurations, present UI rooms, and trigger macro executions.

Confidence tags follow Section 00 §4.5 (`[CONFIRMED]`, `[STATIC]`, `[INFERRED]`).

---

## 2. Client Lifecycle & Architecture

```
   +-----------------------------------------------------------+
   | 1. Discovery (UDP 61510)                                  |
   | - Broadcast PACKET_FIND (0xC350)                          |
   | - Receive PACKET_FIND_ACK (0xC353) with controller IP/MAC |
   +-----------------------------+-----------------------------+
                                 |
                                 v
   +-----------------------------------------------------------+
   | 2. Manifest & Package Sync (TCP 51520 & FTP 21)           |
   | - Query GET_VERSION (0x2010) with Client Model & ID       |
   | - Download M<model>_O<id>_C<token>_model.json             |
   | - Download Theme / Graphics packages from updateroot/     |
   +-----------------------------+-----------------------------+
                                 |
                                 v
   +-----------------------------------------------------------+
   | 3. Graphical UI Engine (TCP 51530 / 51533)                |
   | - Connect to rui_svrd.exe using magic header 0xA55A       |
   | - Stream real-time volume, state, and menu updates        |
   +-----------------------------+-----------------------------+
                                 |
                                 v
   +-----------------------------------------------------------+
   | 4. Client Authentication (TCP 51510)                      |
   | - Transmit CLIENT_LOGIN_REQ (0x1389) with Client ID       |
   | - Controller verifies seat in Table 4                     |
   | - Receive CLIENT_LOGIN_ACK (0x138C) on authorization      |
   +-----------------------------+-----------------------------+
                                 |
                                 v
   +-----------------------------------------------------------+
   | 5. Runtime Automation (TCP 51510)                         |
   | - Send MACRO_ACTION (0x07DA) & Room Power commands        |
   | - Receive action execution ACK (0x07DB)                   |
   +-----------------------------------------------------------+

---

## 3. Required Project Configuration Deliverables

To authorize and render a mobile client interface for a project, the programmer software MUST provide three distinct artifacts:

### 3.1 Client Registration in `mcs_info.dat`

The `clients` array in `mcs_info.dat` MUST declare each authorized interface:

| Field | Type | Description |
|---|---|---|
| `obj_id` | integer | Unique client interface object identifier (e.g. `5`) `[CONFIRMED]` |
| `name` | string | Interface display label (e.g. `"iPhone Mobile"`) `[CONFIRMED]` |
| `model` | string | Target model profile: `"IPhoneV2"`, `"IPadV2"`, `"Android Tablet"`, `"TKP-7600"` `[CONFIRMED]` |

Standard Model Identifiers:
- `IPhoneV2`: `E207`
- `IPadV2`: `E208`
- `Android Tablet`: `E206`
- `TKP-7600`: `MB101`

### 3.2 Client Model JSON (`model.json`)

Staged under `/remote/Store/updateroot/` with naming format:
`M<model>_O<client_id>_C<token>_model.json`
Example: `ME207_O0000000000000005_C0013C529_model.json`.

Top-Level Structure:

| Field | Type | Description |
|---|---|---|
| `header.version` | integer | Data schema version (`65536`) `[CONFIRMED]` |
| `header.chksum` | integer | Signed 32-bit container checksum `[STATIC]` |
| `header.type` | string | Header descriptor type (`"Common"`) `[CONFIRMED]` |
| `data.config.obj` | object | Interface operational parameters `[CONFIRMED]` |
| `data.cameras.obj` | object | Configured surveillance camera channels `[CONFIRMED]` |

`data.config.obj` Fields:

| Field | Type | Description |
|---|---|---|
| `id` | integer | Client object ID matching `obj_id` in `mcs_info.dat` `[CONFIRMED]` |
| `name` | string | Client label matching `mcs_info.dat` `[CONFIRMED]` |
| `room` | string | Default selected room name `[CONFIRMED]` |
| `room_id` | integer | Default selected room ID `[CONFIRMED]` |
| `rooms_module.rooms` | array | Integer list of room IDs accessible from this client `[CONFIRMED]` |
| `panel_cnt` | integer | Total number of navigation panels `[CONFIRMED]` |
| `use_notify` | integer | Notification banner enable (`1` or `0`) `[CONFIRMED]` |

### 3.3 Position Record (`*_position.info`)

Staged under `/remote/Store/updateroot/` as `M<model>_O<client_id>_T<token>_position.info`:
Single ASCII line containing:
```
<Room Name>\t<Client Name>\r\n
```

### 3.4 Room Hierarchy, Device Catalog & Sub-Page Navigation (`core_model.json`)

Mobile applications (iOS, Android) and network touchscreens discover the physical room structure, assigned devices, button layouts, and macro bindings by retrieving `core_model.json` from `/remote/Store/updateroot/` over anonymous FTP (TCP port 21) `[CONFIRMED]`.

#### 3.4.1 Staged Filename Grammar & Distribution

- **Staged Filename Pattern:**
  ```
  T{TEMPLATE_ID}_A0000000000000000_C{CHKSUM}_core_model.json
  ```
  where `TEMPLATE_ID` is a 4-digit decimal template identifier:
  - `0002`: Standard UI template (consumed by iPhone `ME207`, Android `ME206`, iPad `ME208`, and 7"/10" in-wall touchscreens TKP-7600 / TDC-7100) `[CONFIRMED]`.
  - `0003`: High-resolution UI template (consumed by high-definition consoles TKP-8600 / TKP-9600) `[CONFIRMED]`.
- **Cardinality:** Exactly one core model data structure is authored per project. It is replicated and staged under each client template identifier supported in the installation `[CONFIRMED]`.

#### 3.4.2 Schema Specification (`data.rooms[]`)

The root document contains a top-level `data` dictionary with `config`, `vars`, `con_devs`, and `rooms`:

```
data
├── config: { obj: { system_name, ... }, chk_obj: int }
├── vars:   { obj: { items: [...] }, chk_obj: int }
├── con_devs: [ { id: int, type: int, obj: { fav: {} } }, ... ]
└── rooms:  [ RoomEntry, ... ]
```

Each element in `data.rooms[]` defines a physical room entity:

| Field | Type | Description |
|---|---|---|
| `id` | Integer | Physical room identifier (0..65535). Used directly as `room_id` in TCP 51510 commands (`0x07DA`, `0x000D`, `0x0012`) and `DnData.dn` Table 10 `[CONFIRMED]`. |
| `info` | Object | Envelope containing room presentation metadata `[CONFIRMED]`. |
| `info.obj.name` | String | User-visible room display name (e.g. `"Living Room"`) `[CONFIRMED]`. |
| `info.obj.room_image` | String | Path or identifier for room background photo (e.g. `"R:1.JPG"`) `[CONFIRMED]`. |
| `info.obj.background` | String | Path or identifier for fallback chrome wallpaper (e.g. `"R:0.JPG"`) `[CONFIRMED]`. |
| `info.obj.status_bar` | Array | 4-element array of status bar indicators `[{"idx": 0..3, "type": 0, "model_id": -1}]` `[CONFIRMED]`. |
| `info.obj.startup_dev` | Integer | Menu device ID selected upon room activation (convention: `20481 + room_id`) `[CONFIRMED]`. |
| `info.obj.main_dev` | Integer | Room main menu container device ID (`20481 + room_id`) `[CONFIRMED]`. |
| `info.obj.lights_dev` | Integer | Assigned lighting device ID or `-1` if unassigned `[CONFIRMED]`. |
| `info.obj.fav_dev` | Integer | Assigned favorites container ID or `-1` if unassigned `[CONFIRMED]`. |
| `info.chk_obj` | Integer | Validation integrity token (e.g. `1067956892`) `[CONFIRMED]`. |
| `menu_devs` | Object | Envelope containing main room menu tile navigation definitions `[CONFIRMED]`. |
| `menu_devs.obj` | Array | Array of menu containers. Menu buttons MUST be padded to multiples of 6 (6 or 12 items) with hidden placeholders `{"id": -1, "name": "", "hidden": 1, "image": "", "macro": []}` `[CONFIRMED]`. |
| `devices` | Array | Array of enveloped controlled device objects assigned to this room `[CONFIRMED]`. |
| `scenes` | Object | Envelope (`obj: []`, `chk_obj: int`) for room automation scenes `[CONFIRMED]`. |

#### 3.4.3 Device Listing & Function Discovery (`rooms[].devices[]`)

Each entry in `rooms[].devices[]` encapsulates a controllable hardware endpoint:

| Device Field | Type | Description |
|---|---|---|
| `obj.id` | Integer | **Device Identifier.** Directly equals `con_devs[].id` in `mcs.dat` and `DeviceObjTable` (Table 5) in `DnData.dn` `[CONFIRMED]`. |
| `obj.name` | String | Display name of the device `[CONFIRMED]`. |
| `obj.model_name` | String | Hardware model classification string `[CONFIRMED]`. |
| `obj.device_type` | String | Device category archetype (e.g. `"DBMODEL"`) `[CONFIRMED]`. |
| `obj.link_btn_id` | Integer | Menu button jump anchor (typically `2048 + device_index`) `[CONFIRMED]`. |
| `obj.hard_btns` | Array | Hard button macro mappings (bottom mobile navigation bar) `[CONFIRMED]`. |
| `obj.pages` | Array | Swipeable UI template pages (`type`, `name`, `id`, `buttons[]`) `[CONFIRMED]`. |
| `obj.quick_bar` | Array | 10-element quick action array `[{"id": 0} .. {"id": 9}]` `[CONFIRMED]`. |
| `obj.keyboard` | Object | Keyboard mapping descriptor `{"con_dev_id": <id>, "keyboard_id": 0}` `[CONFIRMED]`. |
| `chk_obj` | Integer | Integrity token (`1067956892`) `[CONFIRMED]`. |

#### 3.4.4 Button Macro Encoding & Discrete Function Resolution

Because credential-free LAN clients cannot access `mcs.dat` (which is restricted to `/remote/Store/data/`), clients discover discrete function IDs (`funcs[].id`) by parsing the `macro` arrays in `hard_btns[]` and `pages[].buttons[]`:

Every actionable button defines a macro execution step of type 33:
```json
"macro": [{"type": 33, "code": 131347}]
```
The 32-bit `code` integer packs the target device and discrete function:

$$\text{code} = (\text{device\_id} \ll 16) \mid \text{func\_id}$$

Clients decode the discrete function ID using bitwise masking:
- **Target Device:** `device_id = code >> 16`
- **Target Function:** `func_id = code & 0xFFFF`

*Example:* For `code = 131347` (`0x00020113`), `device_id = 2` and `func_id = 275` (`Vol+`).

#### 3.4.5 Standard Page Template Categories

Total Control Mobile view controllers instantiate fixed swipeable page templates:
- `type: 4`, `name: "channel"`: Channel rockers / brightness controls (`278`, `279`, `280`).
- `type: 1`, `name: "numeric keypad"`: 10-key digit pad (`337`–`348`).
- `type: 7`, `name: "guide/menu/info/exit"`: System menu controls (`304`–`307`).
- `type: 2`, `name: "navigation"`: Directional D-Pad (`320`–`324`).
- `type: 6`, `name: "color buttons"`: Red, Green, Yellow, Blue keys (`548`–`551`).
- `type: 3`, `name: "transport"`: Playback controls (`288` Play, `289` Stop, `291` Pause, `292` Rew, `293` Fwd).
- `type: 0`, `name: "additional A"`: Custom function keys mapped to UI IDs `"A"` through `"F"` resolved via `ui_button_table.json`.

#### 3.4.6 Mobile Bottom Navigation Bar Mapping

On mobile client interfaces, fixed hard button slots map to standardized operations:
- `hbtn_id: 4625`: **`Main` button** (center icon) $\rightarrow$ macro `[{"type": 54, "room": <room_id>}]` (jump to room main menu) `[CONFIRMED]`.
- `hbtn_id: 4624`: **`Power` button** (far-right icon) $\rightarrow$ macro executing Room Power Off / state mutation `[CONFIRMED]`.
- `hbtn_id: 275` / `276` / `277`: Volume Up, Volume Down, and Mute `[CONFIRMED]`.

#### 3.4.7 Minimal Room & Device Test Vector (`core_model.json`)

```json
{"data":{"config":{"obj":{"system_name":"Minimal Interop","key":"","ddns":{"domain":"","port":0},"bstation_net":"","system_net":{"dns":"192.168.1.1","dns2":"0.0.0.0","subnet":"255.255.255.0","gateway":"192.168.1.1"},"link_vol_text":"Linked Audio","file_indicator":1730548800},"chk_obj":1067956892},"vars":{"obj":{"items":[]},"chk_obj":1067956892},"con_devs":[{"obj":{"fav":{}},"id":2,"type ":0}],"rooms":[{"id":1,"info":{"obj":{"name":"Living Room","room_image":"R:1.JPG","background":"R:0.JPG","status_bar":[{"idx":0,"type":0,"model_id":-1},{"idx":1,"type":0,"model_id":-1},{"idx":2,"type":0,"model_id":-1},{"idx":3,"type":0,"model_id":-1}],"startup_dev":20482,"main_dev":20482,"lights_dev":-1,"fav_dev":-1},"chk_obj":1067956892},"menu_devs":{"obj":[{"id":20482,"name":"Main","link_btn_id":-1,"hard_btns":[{"id":-1,"hbtn_id":4625,"macro":[{"type":54,"room":1}]},{"id":-1,"hbtn_id":4624,"macro":[{"type":33,"code":196886}]}],"menu_buttons":[{"id":2049,"name":"Living Room TV","image":"1.png","macro":[{"type":33,"code":196886}]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]}]}],"chk_obj":1067956892},"devices":[{"obj":{"id":2,"name":"Living Room TV","model_name":"Living Room TV","type":0,"link_btn_id":2049,"device_type":"DBMODEL","hard_btns":[{"id":-1,"hbtn_id":275,"macro":[{"type":33,"code":131347}]},{"id":-1,"hbtn_id":276,"macro":[{"type":33,"code":131348}]},{"id":-1,"hbtn_id":277,"macro":[{"type":33,"code":131349}]},{"id":-1,"hbtn_id":4625,"macro":[{"type":54,"room":1}]},{"id":-1,"hbtn_id":4624,"macro":[{"type":33,"code":196886}]}],"pages":[{"type":4,"name":"channel","id":0,"buttons":[{"id":279,"btn_type":0,"hbtn_id":279,"name":"CH +","idx":0,"ui_id":"279","macro":[{"type":33,"code":131351}]},{"id":280,"btn_type":0,"hbtn_id":280,"name":"CH -","idx":1,"ui_id":"280","macro":[{"type":33,"code":131352}]},{"id":-1,"btn_type":0,"hbtn_id":278,"name":"","idx":2,"hidden":2,"hidden_text":1,"ui_id":"278","macro":[]}]}],"quick_bar":[{"id":0},{"id":1},{"id":2},{"id":3},{"id":4},{"id":5},{"id":6},{"id":7},{"id":8},{"id":9}],"keyboard":{"con_dev_id":2,"keyboard_id":0}},"chk_obj":1067956892}],"scenes":{"obj":[],"chk_obj":1067956892}}],"camera_popups":{"obj":[],"chk_obj":1067956892},"trfzws":{"obj":[],"chk_obj":1067956892},"notifications":{"obj":[],"chk_obj":1067956892},"door_stations":{"obj":[],"chk_obj":1067956892}},"header":{"version":65536,"chksum":0,"type":"Common","firmware":0}}
```

## 4. UI Server Protocol (TCP Port 51530)

When connecting to `rui_svrd.exe`:
- Every frame begins with a 6-byte binary header:

| Offset | Size | Field | Type | Allowed Values | Description |
|---|---|---|---|---|---|
| 0x00 | 2 | `magic` | uint16 LE | `0xA55A` | Protocol synchronization magic `[CONFIRMED]` |
| 0x02 | 2 | `msg_type` | uint16 LE | Non-zero | Message identifier `[CONFIRMED]` |
| 0x04 | 2 | `total_length`| uint16 LE | `6` to `2048` | Total frame length including 6-byte header `[CONFIRMED]` |
| 0x06 | `total_length - 6` | `payload` | bytes | | Message body |


---

## 5. Client Authentication & Seat Licensing (TCP Port 51510)

When a mobile client connects to the automation control service on TCP 51510, it transmits `CLIENT_LOGIN_REQ` (`0x1389`) carrying its assigned Client ID. `BS.exe` verifies the session against Table 4 and returns `0x138C` (`CLIENT_LOGIN_ACK`, Status=1, AuthID=1) `[CONFIRMED]`.
(Note: While real clients also report download status to `rmslocalsvrd.exe` on TCP 51520 via opcode `0x2070` (`UPD_DOWNLOAD_STATUS`), port 51510 accepts `0x1389` login handshakes independently.)

### 5.1 `CLIENT_LOGIN_REQ` (Opcode `0x1389`)

Transmitted by the client immediately upon establishing connection to TCP 51510:

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| `0x00` | 1 | `flag` | `uint8` | Client packet framing flag (`0x01`) `[CONFIRMED]` |
| `0x01` | 2 | `length` | `uint16_le` | Payload length following byte 2 (`0x000D` = 13 bytes) `[CONFIRMED]` |
| `0x03` | 2 | `opcode` | `uint16_le` | Login request opcode (`0x1389`) `[CONFIRMED]` |
| `0x05` | 2 | `client_id` | `uint16_le` | Client seat object identifier matching `mcs_info.dat` `[CONFIRMED]` |
| `0x07` | 9 | `reserved` | `uint8[9]` | Trailing reserved padding (all zeroes) `[CONFIRMED]` |

### 5.2 `CLIENT_LOGIN_ACK` (Opcode `0x138C`)

Returned by `BS.exe` when the client seat is present and authorized in Table 4 (`ClientTable`):

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| `0x00` | 1 | `flag` | `uint8` | Server packet framing flag (`0x00`) `[CONFIRMED]` |
| `0x01` | 2 | `length` | `uint16_le` | Payload length following byte 2 (`0x0005` = 5 bytes) `[CONFIRMED]` |
| `0x03` | 2 | `opcode` | `uint16_le` | Login ACK opcode (`0x138C`) `[CONFIRMED]` |
| `0x05` | 1 | `status` | `uint8` | Authorization status (`0x01` = Success) `[CONFIRMED]` |
| `0x06` | 2 | `auth_id` | `uint16_le` | Session authorization token assigned by controller `[CONFIRMED]` |

### 5.3 `CLIENT_LOGIN_NACK` (Opcode `0x13EB`)

Returned by `BS.exe` when the requested seat is missing from Table 4 or license validation fails:

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| `0x00` | 1 | `flag` | `uint8` | Server packet framing flag (`0x00`) `[CONFIRMED]` |
| `0x01` | 2 | `length` | `uint16_le` | Payload length following byte 2 (`0x0007` = 7 bytes) `[CONFIRMED]` |
| `0x03` | 2 | `opcode` | `uint16_le` | Login rejection opcode (`0x13EB`) `[CONFIRMED]` |
| `0x05` | 1 | `status` | `uint8` | Status indicator (`0x01`) `[CONFIRMED]` |
| `0x06` | 4 | `reason` | `uint32_le` | Rejection error code (`0x00000000` = Unknown / Unregistered seat) `[CONFIRMED]` |
---

## 6. Error Behavior

1. **Unregistered Client Model:** If a client requests files for an undeclared model ID from `rmslocalsvrd.exe`, the server returns `count = 0`.
2. **Missing `model.json`:** If the client manifest or `model.json` is missing from `updateroot/`, the client fails to populate room views.
3. **Invalid Magic on Port 51530:** If the client frame does not begin with `0xA55A`, `rui_svrd.exe` logs `Packet partial or wrong. ignored` and discards the packet.

---

## 7. Test Vectors

### 7.1 Minimal Client Registration (`mcs_info.dat`)

```json
{"clients":[{"obj_id":5,"name":"Mobile Client","model":"IPhoneV2"}]}
```

### 7.2 Minimal Client Model Descriptor (`model.json`)

```json
{"header":{"version":65536,"chksum":0,"type":"Common","firmware":0},"data":{"config":{"obj":{"id":5,"name":"Mobile Client","room":"Living Room","room_id":0,"rooms_module":{"type":1,"rooms":[0]},"panel_cnt":6,"use_notify":1,"file_indicator":1730548800},"chk_obj":0},"cameras":{"obj":{"channel":0,"cycle_time":0,"items":[]},"chk_obj":0}}}
```

### 7.3 Position Info File

Raw ASCII bytes:
```
Living Room\tMobile Client\r\n
```

---

## 8. Open Items

1. **Intercom Signaling:** SIP intercom channel integration (`5060`) between touchscreens and mobile clients is optional for standard AV control.
