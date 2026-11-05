---
title: "Mobile and Remote Client Protocols"
section: "53-client-protocols"
version: "0.1.0"
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

### 3.4 Device Sub-Page & Navigation Model (`core_model.json`)

To render active device remote views upon menu card selection, the client view controller relies on conventions established across vendor baseline configurations:

1. **Room Menu Device ID Convention**:
   - The room menu container (`menu_devs[].id`) and its matching references (`startup_dev`, `main_dev`) follow the formula `[INFERRED]`:
     $$\text{menu\_dev\_id} = 20481 + \text{room\_id}$$
   - Example: Room 0 uses `20481`; Room 1 uses `20482`. Adopting this convention aligns room navigation targets with client expectation.

2. **Observed Device Envelope Fields**:
   - In baseline projects, each device object under `rooms[].devices[].obj` includes `[INFERRED]`:
     - `quick_bar`: 10-element array `[{"id": 0} .. {"id": 9}]`.
     - `keyboard`: object `{"con_dev_id": <id>, "keyboard_id": 0}`.
     - `hard_btns`: array of hard button mapping objects.

3. **Standard Page Template Categories**:
   - Total Control Mobile view controllers instantiate fixed swipeable page templates `[INFERRED]`:
     - `type: 4`, `name: "channel"`: Channel rockers / brightness controls (`278`, `279`, `280`).
     - `type: 1`, `name: "numeric keypad"`: 10-key digit pad (`337`–`348`).
     - `type: 7`, `name: "guide/menu/info/exit"`: System menu controls (`304`–`307`).
     - `type: 2`, `name: "navigation"`: Directional D-Pad (`320`–`324`).
     - `type: 6`, `name: "color buttons"`: Red, Green, Yellow, Blue keys (`548`–`551`).
     - `type: 3`, `name: "transport"`: Playback controls (`288` Play, `289` Stop, `291` Pause, `292` Rew, `293` Fwd).
     - `type: 0`, `name: "additional A"`: Custom function keys mapped to UI IDs `"A"` through `"F"` resolved via `ui_button_table.json`. Devices with further custom controls chain additional `type: 0` pages (`"additional B"` through `"additional E"`, with 8 to 11 total pages observed on TVs and cable boxes).

4. **Mobile Bottom Navigation Bar Mapping**:
   - On the mobile client interface, hard button identifiers are assigned as:
     - `hbtn_id: 4625`: **`Main` button** (center icon) $\rightarrow$ macro `[{"type": 54, "room": <room_id>}]` (jump to room main menu) `[CONFIRMED]`.
     - `hbtn_id: 4624`: **`Power` button** (far-right icon) $\rightarrow$ macro executing Room Power Off / state mutation `[CONFIRMED]`.
     - `hbtn_id: 275` / `276` / `277`: Volume Up, Volume Down, and Mute `[CONFIRMED]`.
---

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
