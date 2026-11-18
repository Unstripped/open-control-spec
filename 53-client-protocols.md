---
title: "Mobile and Remote Client Protocols"
section: "53-client-protocols"
version: "0.5.2"
status: review
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
   | - Broadcast DISCOVERY_PROBE (0xC350)                      |
   | - Receive DISCOVERY_REPLY (0xC353) with controller IP/MAC |
   +-----------------------------+-----------------------------+
                                 |
                                 v
   +-----------------------------------------------------------+
   | 2. Manifest & Package Sync (TCP 51520 & FTP 21)           |
   | - Query VERSION_QUERY (0x2010) with Client Model & ID     |
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
   | - Transmit CLIENT_LOGIN_REQUEST (0x1389) with Client ID   |
   | - Controller verifies seat in Table 4                     |
   | - Receive CLIENT_LOGIN_ACCEPT (0x138C) on authorization   |
   +-----------------------------+-----------------------------+
                                 |
                                 v
   +-----------------------------------------------------------+
   | 5. Runtime Automation (TCP 51510)                         |
   | - Send MACRO_TRIGGER (0x07DA) & Room Power commands       |
   | - Parse opcode-specific replies; see section 52            |
   +-----------------------------------------------------------+

---

## 3. Required Project Configuration Deliverables

To register and render a mobile client interface for a project, the programmer software MUST provide three distinct artifacts:

### 3.1 Client Registration in `mcs_info.dat`

The `clients` array in `mcs_info.dat` MUST declare each registered interface:

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
  - `0002`: phone UI content set. In a typical installation the iPhone client model (`E207`) manifest names the `T0002` file `[CONFIRMED]`.
  - `0003`: tablet UI content set. In a typical installation the Android tablet (`E206`) and iPad (`E208`) manifests name the `T0003` file `[CONFIRMED]`.
  - Use by in-wall touchscreens and consoles is not observed.
- **Cardinality:** The programmer authors one core model per project and stages a copy under each template identifier. The two copies are typically byte-identical `[CONFIRMED]`. Clients MUST NOT assume that copies are identical.
- **Selecting the active file:** A client MUST load the single `core_model.json` named in its own model-wide manifest `M<client model>_A0000000000000000_T<t>.info` (`20-upload-manifest` §4.3). It MUST NOT choose a file by matching names in the staging directory. Several staged `core_model.json` files are normal: one per template identifier, and stale copies with other `C` tokens may remain until the controller removes superseded files (`20-upload-manifest` §6.3). A file that no manifest names is not part of the active project.

#### 3.4.2 Schema Specification (`data.rooms[]`)

The root document contains a top-level `data` dictionary with `config`, `vars`, `con_devs`, and `rooms`:

```
data
├── config: { obj: { system_name, ... }, chk_obj: int }
├── vars:   { obj: { items: [...] }, chk_obj: int }
├── con_devs: [ { id: int, "type ": int, obj: { fav: {} } }, ... ]
└── rooms:  [ RoomEntry, ... ]
```

**`vars`.** `vars.obj.items` is an array of bare JSON integers, not objects. An item carries no name, type, or value. In every observed project the array is empty `[CONFIRMED]`, and no observed client reads its contents `[STATIC]`. Whether a non-empty array lists project variable IDs is not established `[INFERRED]`. Clients MUST accept any array of integers, including a non-empty one, and SHOULD NOT depend on its contents. Generators SHOULD emit an empty array and MUST NOT emit objects; a reference client rejects the whole model if an item is not a number `[STATIC]`. Variable display data is bound through room scene entries (`rooms[].scenes.obj[]`), not through `vars`: `var_id` is the controller variable ID (the same `n` as MQTT `variable_id_{n}`), and `states[]` holds per-value display entries `{name, image_name, val}` `[STATIC]`. Older projects use a `type: 2` scene entry with `scene_n` / `scene_p` fields `[STATIC]`. Runtime variable values are read with `VARIABLE_TABLE_QUERY` (`52-control-protocol` §3.6). Controller daemons do not parse `core_model.json`; only clients read it `[STATIC]`.

**`con_devs[]` and the `"type "` key.** The key really is spelled `"type "`, with one trailing space (`0x74 0x79 0x70 0x65 0x20`). All observed entries use this spelling `[CONFIRMED]`. Generators MUST emit `"type "`. Readers MUST look up `"type "` and SHOULD also accept `"type"`. Every observed value is `0`, and the meaning of the value is **unresolved**. In observed projects, `con_devs[].id` values are connected-device ids, the same ids as `mcs.dat` `con_devs[].id`. The list covers only the devices that have UI pages: a multi-zone amplifier device is typically absent `[CONFIRMED]`.

**Other top-level keys.** The reference file also contains `camera_popups`, `trfzws`, `notifications`, and `door_stations` (envelopes with `obj` arrays) `[CONFIRMED]`. Clients MUST ignore keys they do not understand.

Each element in `data.rooms[]` defines a physical room entity:

| Field | Type | Description |
|---|---|---|
| `id` | Integer | Physical room identifier (0..65535). Used directly as `room_id` in TCP 51510 room commands (`0x000D` room power off, `0x0012` power-menu query) and in `DnData.dn` Table 10 `[CONFIRMED]`. `0x07DA` instead carries a project trigger code; see [§3.1 of the control protocol](52-control-protocol.md). |
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
| `obj.id` | Integer | **UI object identifier**, unique across the project. It is **not** a connected-device id (`mcs.dat` `con_devs[].id`) `[CONFIRMED]`. It is the high half of every `code` on this device's buttons (§3.4.4). One physical device that appears in two rooms gets two different object ids. Example: UI objects `7` and `9`, in different rooms, both control connected device `3`; UI object `8` controls connected device `4`, while connected device `8` is a different device `[CONFIRMED]`. |
| `obj.name` | String | Display name of the device `[CONFIRMED]`. |
| `obj.model_name` | String | Hardware model classification string `[CONFIRMED]`. |
| `obj.device_type` | String | Device category archetype (e.g. `"DBMODEL"`) `[CONFIRMED]`. |
| `obj.link_btn_id` | Integer | Menu button jump anchor (typically `2048 + device_index`) `[CONFIRMED]`. |
| `obj.hard_btns` | Array | Hard button macro mappings (bottom mobile navigation bar) `[CONFIRMED]`. |
| `obj.pages` | Array | Swipeable UI template pages (`type`, `name`, `id`, `buttons[]`) `[CONFIRMED]`. |
| `obj.quick_bar` | Array | 10-element quick action array `[{"id": 0} .. {"id": 9}]` `[CONFIRMED]`. |
| `obj.keyboard` | Object | Keyboard mapping descriptor `{"con_dev_id": <id>, "keyboard_id": 0}`. Observed projects have `con_dev_id` equal to the connected device that the room object controls `[CONFIRMED]`. Clients MAY use it to map a room object to a connected device `[INFERRED]`. |
| `chk_obj` | Integer | Integrity token (`1067956892`) `[CONFIRMED]`. |

**Multi-zone amplifiers (HDA).** A multi-zone amplifier is not a room device. In observed projects, a multi-zone amplifier does not appear in any `rooms[].devices[]` or in `data.con_devs[]` `[CONFIRMED]`. A room reaches its amplifier zone through the controller's room audio routing. The room's menu hard buttons and the device pages without their own volume buttons carry `275`/`276`/`277` codes; the controller applies them to the zone linked to the room (`32-download-database` §5.3, routing descriptor and volume punch-through) `[STATIC]` (structure) / `[INFERRED]` (runtime routing; not live-tested). Amplifier source switching is not exposed as named client buttons. In the reference, input changes are steps inside the controller-side macro of each room menu tile, and a client can select a source by triggering the tile's `code` `[CONFIRMED]` (macro contents). A client can also route an input directly with `0x0132`; `core_model.json` does not say which amplifier inputs have a bound source, and the controller lists them in reply to `0x0130` (`52-control-protocol` §3.11.8, §3.11.12) `[CONFIRMED]`. For direct per-room zone volume and mute, see `52-control-protocol`.

#### 3.4.4 Macro Steps & Button Code Encoding

Each `macro` array lists steps in execution order. A step's `type` uses the same number as the controller database action opcode (`32-download-database` §5.4.2). Observed projects contain only three step types: 33, 4, and 54 `[CONFIRMED]`.

| `type` | Keys | Meaning | Over the network |
|---|---|---|---|
| `33` | `code` | Trigger a controller-side button | The client sends `code` to the controller as a macro trigger (`MACRO_TRIGGER`, `52-control-protocol` §3.1). The controller runs the matching button's action list from its database `[STATIC]`. |
| `4` | `j_type`, `target` | Navigate the client UI (jump) | **Client-local.** No controller command runs a jump step. A networked client implements it as navigation, or ignores it if it has no such screen `[STATIC]`. |
| `54` | `room` | Turn the room off | The client sends `ROOM_POWER_OFF` (`0x000D`) with `room_id = room` `[STATIC]`. A client MAY open a local room-power page on press-and-hold; this is client-local behavior. |

Clients MUST skip step types they do not implement and continue with the next step.

**Type 33 `code`.** The 32-bit `code` packs a UI object id and a button id:

$$\text{code} = (\text{object\_id} \ll 16) \mid \text{button\_id}$$

- `object_id = code >> 16` is a project-wide UI object id. It is **not** a connected-device id. On device pages it equals that room device's `obj.id` (§3.4.3). On a room menu (`menu_devs`) it is the room's controller-side menu object id. That id is different from `menu_devs.obj[].id` (`20481 + room_id`). The reference uses menu objects `1`, `3`, and `5` for rooms 0, 1, and 2; lighting objects `2`, `4`, and `6`; and device objects `7`–`13` `[CONFIRMED]`.
- `button_id = code & 0xFFFF`. On device pages a programmer sets it to the controlled device's discrete function id (`mcs.dat` `funcs[].id`). Example: reference object `7`, button `279` sends function `279` to connected device `3` `[CONFIRMED]`. Clients SHOULD treat `code` as an opaque trigger and SHOULD NOT build codes for functions that no button carries.

*Example:* `code = 131347` (`0x00020113`) is object `2`, button `275` (`Vol+`).

**Type 4 jump.** A jump is client-local navigation; it is never sent to the controller as a command. `j_type` selects the jump kind and `target` selects the destination. Jump kinds are numbered: `0` = default navigation, `1` = page, `2` = device, `3` = sub-menu, `4` = special screen `[STATIC]`. Transition styles are numbered `0` = replace, `1` = fly from right, `2` = from left, `3` = from top, `4` = from bottom `[STATIC]`.

For `j_type = 0`, `target` selects a built-in destination: `0` = return to the current/last device, `1`/`2` = previous/next page, `3`/`4` = previous/next navigation page, `5`/`6` = previous/next device, `7` = room main menu, `8` = room list `[STATIC]`. Values above `8` are undefined: reference clients take no action for them and the controller does not interpret them `[STATIC]`. A client MUST treat an undefined `target` as a no-op and continue with the next step. Hard button `4640` is bound in the reference project to `j_type = 0`, `target = 9` `[CONFIRMED]`; that binding has no defined effect, and a client MAY omit the button. Only `j_type = 0` is observed in `core_model.json` `[CONFIRMED]`.

For `j_type = 4`, the destination is a special-screen code, a separate value space from the default-jump targets (code `9` here means navigation page, unrelated to default target `9`) `[STATIC]`: `0` now playing, `1` wait, `2` sensor, `3` 12 V, `4` info, `5` bass/treble, `6` lighting, `7` camera, `8` number pad page, `9` navigation page, `10` transport page, `11` off-site, `12` two-way lighting, `13` other page, `16` event, `17` alarm, `18` sleep timer, `19` vacation, `21` power, `48` TSP-2000, `49` photo viewer, `64` TRF Z-Wave, `65`/`66` LG lighting/scenes, `67` room scenes, `68` unified module, `69` intercom, `70` ring module, `72` rooms menu, `80` favorite, `81` keyboard, `96` HDA EQ, `97` HDA volume, `98` HDA volume mixer, `99` HDA announcement, `100` HDA input status, `101` HDA amp status, `102` HDA zone status, `103` HDA zone EQ. A client MUST treat a special-screen code it does not implement as a no-op.

A menu-tile jump to a device is not a client step: it lives in the controller-side action list of the tile's type-33 button. After the client triggers the tile, the controller sends the jump to the client `[STATIC]`. That push channel and its payload are **unresolved** in this specification.

#### 3.4.5 Standard Page Template Categories

Clients render a fixed set of swipeable page templates:
- `type: 4`, `name: "channel"`: Channel rockers / brightness controls (`278`, `279`, `280`).
- `type: 1`, `name: "numeric keypad"`: 10-key digit pad (`337`–`348`).
- `type: 7`, `name: "guide/menu/info/exit"`: System menu controls (`304`–`307`).
- `type: 2`, `name: "navigation"`: Directional D-Pad (`320`–`324`).
- `type: 6`, `name: "color buttons"`: Red, Green, Yellow, Blue keys (`548`–`551`).
- `type: 3`, `name: "transport"`: Playback controls (`288` Play, `289` Stop, `291` Pause, `292` Rew, `293` Fwd).
- `type: 0`, `name: "additional A"`: Custom function keys mapped to UI IDs `"A"` through `"F"` resolved via `ui_button_table.json`.

#### 3.4.6 Hard Button Mapping

Observed projects bind hard buttons identically on every room menu and every room device `[CONFIRMED]`:

| `hbtn_id` | Macro | Effect |
|---|---|---|
| `4625` | `[{"type": 4, "j_type": 0, "target": 7}]` | Jump to the room main menu `[CONFIRMED]` (binding) / `[STATIC]` (meaning) |
| `2304` | `[{"type": 4, "j_type": 0, "target": 8}]` | Jump to the room list `[CONFIRMED]` (binding) / `[STATIC]` (meaning) |
| `4640` | `[{"type": 4, "j_type": 0, "target": 9}]` | Undefined target: no defined effect; clients treat it as a no-op `[CONFIRMED]` (binding) / `[STATIC]` (no-op) |
| `4624` | `[{"type": 54, "room": <this room's id>}]` | Turn this room off: `0x000D` with that room id `[CONFIRMED]` (binding) / `[STATIC]` (wire effect) |
| `275` / `276` / `277` | type 33 | Volume up, volume down, and mute for the object's audio (§3.4.3) `[CONFIRMED]` |

In the vendor's iPhone app, `4625` is presented as the main-menu button and `4624` as the room-power button `[CONFIRMED]` (live test). The control protocol has no room power-on opcode; `0x0012` is a read-only power-menu query ([§3.5 of the control protocol](52-control-protocol.md)). A client that needs to switch a room on triggers one of the room's activity buttons with `MACRO_TRIGGER` (`0x07DA`) ([§3.1](52-control-protocol.md)). No hard button in the reference binds room power-on.

#### 3.4.7 Minimal Room & Device Test Vector (`core_model.json`)

```json
{"data":{"config":{"obj":{"system_name":"Minimal Interop","key":"","ddns":{"domain":"","port":0},"bstation_net":"","system_net":{"dns":"192.168.1.1","dns2":"0.0.0.0","subnet":"255.255.255.0","gateway":"192.168.1.1"},"link_vol_text":"Linked Audio","file_indicator":1730548800},"chk_obj":1067956892},"vars":{"obj":{"items":[]},"chk_obj":1067956892},"con_devs":[{"obj":{"fav":{}},"id":2,"type ":0}],"rooms":[{"id":1,"info":{"obj":{"name":"Living Room","room_image":"R:1.JPG","background":"R:0.JPG","status_bar":[{"idx":0,"type":0,"model_id":-1},{"idx":1,"type":0,"model_id":-1},{"idx":2,"type":0,"model_id":-1},{"idx":3,"type":0,"model_id":-1}],"startup_dev":20482,"main_dev":20482,"lights_dev":-1,"fav_dev":-1},"chk_obj":1067956892},"menu_devs":{"obj":[{"id":20482,"name":"Main","link_btn_id":-1,"hard_btns":[{"id":-1,"hbtn_id":4625,"macro":[{"type":4,"j_type":0,"target":7}]},{"id":-1,"hbtn_id":4624,"macro":[{"type":54,"room":1}]}],"menu_buttons":[{"id":2049,"name":"Living Room TV","image":"1.png","macro":[{"type":33,"code":67585}]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]},{"id":-1,"name":"","hidden":1,"image":"","macro":[]}]}],"chk_obj":1067956892},"devices":[{"obj":{"id":2,"name":"Living Room TV","model_name":"Living Room TV","type":0,"link_btn_id":2049,"device_type":"DBMODEL","hard_btns":[{"id":-1,"hbtn_id":275,"macro":[{"type":33,"code":131347}]},{"id":-1,"hbtn_id":276,"macro":[{"type":33,"code":131348}]},{"id":-1,"hbtn_id":277,"macro":[{"type":33,"code":131349}]},{"id":-1,"hbtn_id":4625,"macro":[{"type":4,"j_type":0,"target":7}]},{"id":-1,"hbtn_id":4624,"macro":[{"type":54,"room":1}]}],"pages":[{"type":4,"name":"channel","id":0,"buttons":[{"id":279,"btn_type":0,"hbtn_id":279,"name":"CH +","idx":0,"ui_id":"279","macro":[{"type":33,"code":131351}]},{"id":280,"btn_type":0,"hbtn_id":280,"name":"CH -","idx":1,"ui_id":"280","macro":[{"type":33,"code":131352}]},{"id":-1,"btn_type":0,"hbtn_id":278,"name":"","idx":2,"hidden":2,"hidden_text":1,"ui_id":"278","macro":[]}]}],"quick_bar":[{"id":0},{"id":1},{"id":2},{"id":3},{"id":4},{"id":5},{"id":6},{"id":7},{"id":8},{"id":9}],"keyboard":{"con_dev_id":2,"keyboard_id":0}},"chk_obj":1067956892}],"scenes":{"obj":[],"chk_obj":1067956892}}],"camera_popups":{"obj":[],"chk_obj":1067956892},"trfzws":{"obj":[],"chk_obj":1067956892},"notifications":{"obj":[],"chk_obj":1067956892},"door_stations":{"obj":[],"chk_obj":1067956892}},"header":{"version":65536,"chksum":0,"type":"Common","firmware":0}}
```

In this vector, room device object `2` controls connected device `2`. The two ids are equal only by choice; real projects number them independently (§3.4.3). The menu tile code `67585` = `(1 << 16) | 2049` assumes that room 1's controller-side menu object id is `1`. A generator MUST use the menu object id it writes into the controller database.

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

## 5. Client Login & Seat Registration (TCP Port 51510)

When a mobile client connects to the automation control service on TCP 51510, it transmits `CLIENT_LOGIN_REQUEST` (`0x1389`) carrying its assigned Client ID. `BS.exe` checks the seat against Table 4 and returns `0x138C` (`CLIENT_LOGIN_ACCEPT`, Status=1, AuthID=1) `[CONFIRMED]`.
(Note: While real clients also report download status to `rmslocalsvrd.exe` on TCP 51520 via opcode `0x2070` (`DOWNLOAD_STATUS_REPORT`), port 51510 accepts `0x1389` login handshakes independently.)

### 5.1 `CLIENT_LOGIN_REQUEST` (Opcode `0x1389`)

Transmitted by the client immediately upon establishing connection to TCP 51510:

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| `0x00` | 1 | `flag` | `uint8` | Client packet framing flag (`0x01`) `[CONFIRMED]` |
| `0x01` | 2 | `length` | `uint16_le` | Payload length following byte 2 (`0x000D` = 13 bytes) `[CONFIRMED]` |
| `0x03` | 2 | `opcode` | `uint16_le` | Login request opcode (`0x1389`) `[CONFIRMED]` |
| `0x05` | 2 | `client_id` | `uint16_le` | Client seat object identifier matching `mcs_info.dat` `[CONFIRMED]` |
| `0x07` | 9 | `reserved` | `uint8[9]` | Trailing reserved padding (all zeroes) `[CONFIRMED]` |

### 5.2 `CLIENT_LOGIN_ACCEPT` (Opcode `0x138C`)

Returned when the client seat is present in Table 4 (client table):

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| `0x00` | 1 | `flag` | `uint8` | Server packet framing flag (`0x00`) `[CONFIRMED]` |
| `0x01` | 2 | `length` | `uint16_le` | Payload length following byte 2 (`0x0005` = 5 bytes) `[CONFIRMED]` |
| `0x03` | 2 | `opcode` | `uint16_le` | Login ACK opcode (`0x138C`) `[CONFIRMED]` |
| `0x05` | 1 | `status` | `uint8` | Authorization status (`0x01` = Success) `[CONFIRMED]` |
| `0x06` | 2 | `auth_id` | `uint16_le` | Session authorization token assigned by controller `[CONFIRMED]` |

### 5.3 `CLIENT_LOGIN_REJECT` (Opcode `0x13EB`)

Returned when the requested seat is not registered in Table 4:

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
3. **Invalid Magic on Port 51530:** If the client frame does not begin with `0xA55A`, `rui_svrd.exe` discards the packet.

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
