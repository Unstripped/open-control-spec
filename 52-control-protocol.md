---
title: "Runtime Control Protocol"
section: "52-control-protocol"
version: "0.11.0"
status: review
---

# Runtime Control Protocol

## 1. Scope

The runtime control protocol operates over TCP port 51510 on the primary automation controller. It is used by user interfaces, remotes, test tools, and programmers to query system status, synchronize variables, and trigger macro and device actions.

Connection lifetime is opcode-dependent. Clients MUST follow §4 rather than assuming that every response leaves the connection reusable.

Confidence tags: `[CONFIRMED]` = observed on a live controller; `[STATIC]` = established from controller firmware behavior but not reproduced live; `[INFERRED]` = interpretation consistent with evidence but not directly established.

---

## 2. Wire Framing & Protocol Header

### 2.1 Request Frame Format

Every request begins with a 3-byte framing prefix followed by the command opcode and payload:

| Offset | Size | Field | Type | Allowed Values | Description |
|---|---|---|---|---|---|
| 0x00 | 1 | `frame_type` | uint8 | `0x02` (standard), `0x00` | Frame initiation marker. `0x02` indicates standard framed request; `0x00` indicates legacy direct unproxied request `[CONFIRMED]`. `0x01` is also accepted for `0x07DA` and behaves exactly as `0x02` (§3.1.4) `[CONFIRMED]`. Clients MUST send `0x02` |
| 0x01 | 2 | `length` | uint16 LE | `≥ 2` | Length of subsequent payload bytes including `opcode` `[CONFIRMED]` |
| 0x03 | 2 | `opcode` | uint16 LE | Listed in §3 | Command opcode identifier `[CONFIRMED]` |
| 0x05 | `length - 2` | `payload` | bytes | | Command-specific argument data `[CONFIRMED]` |

Total wire size = `length + 3` bytes (1-byte `frame_type` + 2-byte `length` + `length` bytes of payload).

Payload length checking is opcode-specific. Some opcodes ignore surplus or missing payload bytes; others reject a wrong length with the generic negative response (§2.3). Clients MUST send exactly the payload size documented for each opcode.

### 2.1.1 Request Prefix Byte

The `MACRO_TRIGGER` (`0x07DA`) and `SET_LEVEL_ABSOLUTE` (`0x07ED`) payloads begin with a 1-byte `prefix`. Their command fields start at payload offset `0x01` `[CONFIRMED]`. The commands of §3.11.3–§3.11.9 carry the same byte `[CONFIRMED]` (`0x0144`: `[STATIC]`); for the §3.11.10 opcodes it is `[STATIC]` or not established. Whether other opcodes carry it is not established.

- Clients MUST send `prefix = 0x00`.
- Values `0x20`–`0x3F` select a separate session-establishment path. For any such value, with either opcode, the controller does not perform the command. It replies with the fixed 54-byte frame `00 33 00 72 17 22` followed by a 48-byte body, then closes the connection `[CONFIRMED]`. `0x1772` is not a reply to the command, and that path is outside this specification.
- All other values tested had no observed effect: the full range for `0x07ED`, and `0x00`–`0x04`, `0x08`, `0x10`, `0x40`, and `0x80` for `0x07DA` `[CONFIRMED]`.

A client that omits the prefix shifts every field by one byte. The trigger code's low byte is then read as `prefix`, so a `button_id` whose low byte is `0x20`–`0x3F` produces the `0x1772` frame described above.

### 2.2 Response Frame Format

Responses from the controller return a 3-byte header prefix, response opcode, status byte, and returned data:

| Offset | Size | Field | Type | Observed Values | Description |
|---|---|---|---|---|---|
| 0x00 | 1 | `resp_type` | uint8 | `0x00`, `0x01` | Response category marker. The value is fixed per response path, not per outcome (for example, the `0x07DB` sent for `0x07DA` uses `0x01` and the `0x07DB` sent for `0x000E` uses `0x00`; `0x07EE` and `0xFFFD` use `0x00`; `0xFFFE` is sent with either value depending on its source, §2.3). Clients MUST NOT use it to decide success `[CONFIRMED]` |
| 0x01 | 2 | `resp_length` | uint16 LE | `≥ 3` | Response payload size including `resp_opcode` and `status` `[CONFIRMED]` |
| 0x03 | 2 | `resp_opcode` | uint16 LE | | Response opcode `[CONFIRMED]` |
| 0x05 | 1 | `status` | uint8 | `0x01`, `0x02`, `0x22` (§2.1.1 only) | See below `[CONFIRMED]` |
| 0x06 | `resp_length - 3` | `body` | bytes | | Returned query results or transaction metadata `[CONFIRMED]` |

Total wire size = `resp_length + 3` bytes.

**Status byte.** The meaning of `status` is defined per response opcode, not globally:

- Every `0xFFFD` frame on this port carries `0x01` `[STATIC]`; no `0xFFFD` frame with another status has been observed `[CONFIRMED]` (§3.7).
- `0xFFFE` carries `0x01` or `0x02` depending on its source (§2.3) `[CONFIRMED]`.
- The room and zone mute commands report success as `resp_opcode = 0x0003` with status `0x02` (§3.11.5); the other success frames in this specification carry `0x01` `[CONFIRMED]`.
- `0x00` is not sent with any response opcode defined in this specification `[STATIC]`. It occurs only on a programmer version reply outside its scope.
- `0x22` occurs only on the session-establishment path of §2.1.1 (`00 33 00 72 17 22`), outside this specification `[CONFIRMED]`.

The outcome of a request is conveyed by `resp_opcode`. Clients MUST treat `0xFFFE` as a negative result and `0xFFFD` as a positive result regardless of `status`, and SHOULD treat a status value not documented for the received `resp_opcode` as reserved: it MUST NOT change the outcome the client derives from `resp_opcode`.

Communication is synchronous and half-duplex per connection: a client transmits one framed request and awaits its matching response before issuing any permitted subsequent request. Some requests produce more than one response frame (§3.1); clients MUST parse all frames by `resp_length` and MUST NOT assume one frame per request. The controller never transmits unsolicited asynchronous event frames over this socket; asynchronous status and macro progress events are emitted over UDP 61610/61611, UDP 57002, or MQTT TCP 1883.

### 2.3 Generic Responses (`0xFFFD`, `0xFFFE`)

`0xFFFD` is a generic positive response opcode used by several queries (§3.7–§3.9). `0xFFFE` is a generic negative response opcode. It is not specific to one request. Its first byte (`resp_type`) and its status byte are constants of the code path that sends it, not result codes `[STATIC]`. Sources of `0xFFFE` for the requests in this specification:

| Condition | Frame | When sent | Request performed | Connection effect | Confidence |
|---|---|---|---|---|---|
| Controller has not finished system initialization, or is finishing a project download, when the connection is accepted (§4 rule 7) | `00 03 00 FE FF 01` | On accept, before any request is read | — | Closed about 0.3 s later | `[STATIC]` |
| Another connection from the same client IPv4 address is registered, or 64 client addresses are already registered (§4 rule 10) | `01 03 00 FE FF 01` | Immediately after the first request | No | Closed when the client closes, or after 10 s | `[CONFIRMED]` (same address; not performed) / `[STATIC]` (64-address limit; 10 s close) |
| No complete request arrives within 10,000 ms of connection establishment (§4 rule 1) | `01 03 00 FE FF 01` | 10 s after connecting | — | Closed about 3 s later | `[CONFIRMED]` |
| `NETWORK_STATUS_QUERY` while the controller is not online (§3.7) | `00 03 00 FE FF 01` | Reply | — | Terminal | `[STATIC]` |
| Room-routed level or trigger level command fails (`0x00A2`, `0x001B`, §3.11) | `01 03 00 FE FF 01` | Reply | No | Terminal | `[STATIC]` |
| Room or zone mute or level command names an unknown room, or a device that does not support the command (§3.11.5, §3.11.10) | `00 03 00 FE FF 02` | Reply | No | Terminal | `[CONFIRMED]` |

For `0x07DA`, `0x07ED`, `0x000D`, `0x000E`, `0x0012`, `0x089A`, `0x0FE1`, and `0x0FE6`, a `01 03 00 FE FF 01` that arrives immediately after the request can only be the address rejection `[STATIC]`. Earlier revisions attributed it, for `0x07ED`, to a payload length check; no such check exists (§3.2). Opcodes outside this specification send `0xFFFE` with either first byte on their own errors `[STATIC]`. Clients MUST NOT infer the cause of a `0xFFFE` from its first byte alone; they distinguish the sources by the request sent and by when the frame arrives.

---

## 3. Command Opcode Catalog

### 3.1 Macro Execution (`0x07DA`)

- **Name:** `MACRO_TRIGGER`
- **Request Opcode:** `0x07DA`
- **Request Payload (5 bytes, `length = 7`):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 1 | `prefix` | uint8 | MUST be `0x00` (§2.1.1) `[CONFIRMED]` |
| 0x01 | 4 | `trigger_code` | uint32 LE | Trigger code. The controller splits it into `button_id = trigger_code & 0xFFFF` and `object_id = trigger_code >> 16` `[STATIC]`. For a tested project button, sending the code at offset `0x01` produced device traffic, and sending it at offset `0x00` produced none `[CONFIRMED]`. |

`object_id` is a project-wide user-interface object identifier from the project model (Section 30), and `button_id` identifies a button on that object `[STATIC]`. `object_id` is **not** the `mcs.dat` connected-device identifier; the two numbering spaces differ in reference projects `[STATIC]`. Trigger codes for a project are the button `code` values published by the project model.

Earlier revisions documented a 16-byte payload (`room_id`, `device_id`, `macro_id`, `flags`) and later a 4-byte payload without `prefix`. Neither is correct. Because the acknowledgement does not show whether an action ran (§3.1.2), both forms appear to succeed.

#### 3.1.1 Device Functions, Volume, Mute, and Source Selection

- `MACRO_TRIGGER` carries no zone, flags, or press/hold/release field. A trigger code reaches a zone only through the project's button and macro definitions, and the controller times a press itself (§3.1.4). To address one zone of a multi-zone amplifier directly, clients use the commands in §3.11.
- Function IDs `275` (volume up), `276` (volume down), `277` (mute toggle), `552` (mute on), and `553` (mute off) appear in device function lists for audio zones `[STATIC]`. The mapping from a device function ID to a trigger code is defined by the project model, not by this protocol.
- In reference projects, source selection is performed by controller-side macros attached to room menu tiles, not by a published input function ID `[STATIC]`. Earlier revisions cited function IDs `1001` and `1002` as inputs; those IDs were not observed in any reference project and are withdrawn.
- Absolute level, mute, power, and source for one zone of a multi-zone amplifier are set with the room and amplifier-zone commands in §3.11, each confirmed against a live amplifier. `0x07ED` never reaches an amplifier zone (§3.2).

#### 3.1.2 Responses

The controller sends a sequence of frames on the request's connection `[CONFIRMED]`:

1. `TRIGGER_ACK` (`0x07DB`), 8 bytes: `01 05 00 DB 07 01` followed by a 2-byte body. The body is controller state, not a client-meaningful handle; observed value `01 02` `[CONFIRMED]`. Clients SHOULD ignore its contents.
2. When the trigger code resolves to an action:
   1. `00 04 00 DD 07 01 01` (`resp_opcode = 0x07DD`, body `01`): sent when the action starts.
   2. Zero or more level reports (`0x07E1`, §3.1.3).
   3. `00 04 00 DD 07 01 00` (`0x07DD`, body `00`): sent when the action ends.
   4. `00 03 00 DC 07 01` (`resp_opcode = 0x07DC`, no body).
3. Otherwise `0x07DC` follows `0x07DB` directly. Observed for unknown trigger codes, for trigger code `0`, and for one valid code that produced no observed device traffic `[CONFIRMED]`.

For valid buttons, `0x07DC` arrived about 0.5 s after the request for a mute toggle, 1.0 s for a networked speaker's volume step, and 1.45 s for an amplifier volume press `[CONFIRMED]`. `0x07DC` was the last frame in every observed sequence. Clients MUST keep reading until `0x07DC`; `MACRO_TRIGGER` is terminal after it (§4). Closing the connection earlier does not cancel the action: a client that closed right after `0x07DD` `01` saw the same device commands as one that held the connection open `[CONFIRMED]`. It also does not release the client address, which stays registered until the action ends (§4 rule 10) `[CONFIRMED]`. No frame reports whether the device carried the action out. Clients MUST NOT treat `0x07DC` as an error, and MUST NOT treat `0x07DB` alone as proof that a button ran.

#### 3.1.3 Level Reports (`0x07E1`)

During an action that sends volume or mute commands to a multi-zone amplifier, the controller sends one `0x07E1` frame after each command: `00 0E 00 E1 07 01` followed by an 11-byte body `[CONFIRMED]`. None was sent during volume actions on a networked speaker `[CONFIRMED]`.

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| `0x00` | 1 | `min_level` | uint8 | Observed `0`; consistent with the zone's minimum level `[INFERRED]` |
| `0x01` | 1 | `max_level` | uint8 | Observed `100`; consistent with the zone's maximum level `[INFERRED]` |
| `0x02` | 1 | `level` | uint8 | Zone level after the command; matched the level the amplifier reported `[CONFIRMED]` |
| `0x03` | 1 | `mute` | uint8 | `0` = unmuted, `1` = muted `[CONFIRMED]` |
| `0x04` | 7 | — | bytes | Observed `02 00 00 00 01 00 00` in every report; meaning not established |

Clients SHOULD take `level` from the last report in a sequence as the resulting level.

#### 3.1.4 Volume Steps and Press Hold

A volume button sent through `0x07DA` is a press that the controller holds and releases itself, not a single step. During the hold it sends the device's volume command repeatedly `[CONFIRMED]`.

1. **Send schedule** `[STATIC]`: when the trigger resolves to a device command, the controller sends it immediately and marks the press as held. While the press is held, it sends the command again once more than `ramp_start` ms have elapsed since the previous send, and after that each time more than `ramp_speed` ms have elapsed since the previous send. It checks the press about every 20 ms between sends. The settings come from the command record in the controller's compiled database (`32-download-database` §5.2.1), or from the copy carried by a device-command macro step that names the command directly:
   - repeat option `0`: repeat while held; after release, keep sending at the same cadence until at least `minimum count` sends have been made, then stop;
   - repeat option non-zero: send until `repeat count` sends have been made, then stop.
2. **Release:** the press counts as held until either:
   - the client sends any byte on the request's connection during the action; the controller releases the press at its next check, 8–33 ms later `[CONFIRMED]`; or
   - 1000 ms have elapsed since the press began without a button-status report `[STATIC]`. A LAN client cannot send a report that extends the hold (item 6) `[CONFIRMED]`, so for a LAN client the hold ends at the first byte it sends or at this 1000 ms timeout; the observed six sends fit the timeout `[INFERRED]`.

   Closing the connection does not release the press or cancel the action (§3.1.2), but it can delay a repeat: after a client closed right after `0x07DB`, the press made five sends, one after a 382 ms gap `[CONFIRMED]`.
3. **Observed timing** `[CONFIRMED]`: in the reference project, one volume-up or volume-down press on a device page whose volume is bound to the room's amplifier zone produced six sends, about +16, +486, +682, +876, +1076, and +1274 ms after the request: the first repeat about 465–470 ms after the first send (495 ms in one run), then a period of 192–201 ms. `0x07DC` arrived about 1.45 s after the request. The reference amplifier moves 2 levels per relative command, so one press changed the level by 12. A mute toggle on the same page sent one command. The reference amplifier's volume commands carry repeat option `0`, minimum count `1`, repeat count `1`, ramp start `300`, and ramp speed `100` `[STATIC]`.
4. **Why the period exceeds `ramp_speed`** `[INFERRED]`: each send blocks the action until the device exchange and its `0x07E1` report (§3.1.3) are complete, about 160 ms against the reference amplifier. The observed periods fit `send time + ramp_start + check interval` (≈ 466 ms) for the first repeat and `max(ramp_speed, send time) + check interval` (≈ 198 ms) after it. A slower device or network lengthens every period and can reduce the number of sends within the 1000 ms hold. The send count is therefore a result of controller and device timing, not a project constant.
5. **Not visible to clients:** the repeat and ramp fields exist only in the compiled database (`32-download-database` §5.2.1). They do not appear in `core_model.json`, `mcs.dat`, or any other client-visible model file `[STATIC]`. The size of each step belongs to the device; neither this protocol nor the project sets it. Clients MUST NOT assume a fixed step count or step size, and SHOULD read the resulting level from `0x07E1` reports (§3.1.3) where the device provides them.
6. **A client can shorten a hold but not extend it** `[CONFIRMED]`:
   - Frame type `0x01` is accepted for `0x07DA` and behaves exactly as `0x02`. It does not enable any hold-extension mode.
   - A `0x07DE` button-status frame sent during the action, including one meant to report "still held" (`01 04 00 DE 07 00 01` or `02 04 00 DE 07 00 01`), releases the press like any other byte. The controller sends no acknowledgement for it. The full layout of `0x07DE` is not established.
   - Any byte written before the first repeat limits the press to one send.

   Clients MUST NOT write to a `0x07DA` connection during an action unless they intend to end the press.
7. **Holding a volume button:** a client that wants a sustained volume change SHOULD send discrete presses one after another, one connection at a time, reading each to `0x07DC` before opening the next connection (§4 rule 10). Where the zone is reachable by `0x00A2` or `0x001B` (§3.11), clients SHOULD prefer those: they set an exact level, or move one device step, per request.

### 3.2 Absolute Volume Control (`0x07ED`)

- **Name:** `SET_LEVEL_ABSOLUTE`
- **Request Opcode:** `0x07ED`
- **Description:** Asks the controller to set an absolute level on the connected device bound to a trigger code. It takes effect only for a narrow class of devices (below) and never on a multi-zone amplifier zone `[STATIC]`; against the reference amplifier it sent nothing `[CONFIRMED]`. For amplifier zones clients use `0x001B` or `0x00A2` (§3.11).
- **Step 1, binding resolution** `[STATIC]`: with `object_id = trigger_code >> 16` and `button_id = trigger_code & 0xFFFF`, the controller tries the following in order and uses the first that yields a binding `{connected device, database type, zone index, database index}`:
  1. a room menu whose menu ID equals `object_id` (`32-download-database` §5.3.1): if the menu has punch-through enabled, the punch slot selected by `button_id` (table below), provided that slot names a connected device; otherwise the menu button whose ID is `button_id`, resolved through its macro (item 4);
  2. a device object with ID `object_id` in any room (`32-download-database` §5.3.2): its punch slot as in item 1, otherwise its button `button_id` resolved through its macro;
  3. a module-device entry with ID `object_id` and at least one binding (`32-download-database` §5.3.3): its bindings, used as a punch-through table;
  4. a button resolves through its macro only when the macro's first action is a device-command step (`OP_DEV_CMD`, `0x21`, `32-download-database` §5.4.2). The binding is that step's connected device, database type, zone index, and database index. Any other first action fails, and later actions are not examined.

  | Punch slot | `button_id` values |
  |---|---|
  | power | 273–274 |
  | volume | 275–277 |
  | channel | 278–280 |
  | navigation | 304–307, 320–324, 548–551 |
  | transport | 288, 289, 291–296 |
  | number pad | 336–346, 348 |

  Any other `button_id` (including `530`, `552`, and `553`) selects no slot and falls through to the button's own macro. A volume button such as Vol+ (`275`) is therefore a valid trigger: its `button_id` only selects the volume slot `[STATIC]`. `0x001B` uses the same resolution, and with room-menu, device-object, and module-device volume triggers it reached the reference amplifier zone (§3.11.4) `[CONFIRMED]`.
- **Step 2, device action** `[STATIC]`: for the resolved connected device (Table 5, `32-download-database` §5.3) the controller does one of the following:
  1. **Two-way script:** if the device's two-way script is running, and either its 2-Way target flag and feedback type agree (both IP or both serial) or it has no 2-Way target and its command transport matches its feedback type (Network with IP feedback, or Serial with serial feedback), the controller passes a set-volume request with the binding's zone index and `current_value` to that script. The script is started only for a device whose master script name is at least 2 bytes long.
  2. **Discrete level command:** otherwise, it looks up the command whose ID is `0x1100 + current_value` (4352 + level) in the bound zone's command list and sends it. If the device has no such command, nothing is sent.

  Only `trigger_code` and `current_value` are used; `min_value`, `max_value`, and `mute_status` are echoed and otherwise ignored `[STATIC]`.
- **Which bindings accept it:** `0x07ED` takes effect only when the trigger resolves (step 1) **and** the bound device either runs a two-way script that meets the condition in step 2 or defines a device command with ID `0x1100 + level` `[STATIC]`. A multi-zone amplifier zone never accepts it: the amplifier has no master script, and its database drives levels with JSON command templates, not `0x1100 + n` commands `[STATIC]`. Against the reference amplifier, `0x07ED` with the room-menu and device-object volume triggers that `0x001B` resolves to the amplifier zone, and with the matching volume-down trigger, sent nothing to the amplifier and left the level unchanged `[CONFIRMED]`. No device in the reference installation meets either condition `[STATIC]`.
- **Acceptance is not observable:** `core_model.json` does not carry punch-through bindings, script names, feedback types, or device command IDs, and the `0x07EE` echo (below) is sent whatever the outcome `[STATIC]`; it was identical for resolved triggers that sent nothing and for a trigger that does not exist `[CONFIRMED]`. A client therefore cannot determine from the project model or from the reply whether a trigger accepts `0x07ED`. Clients SHOULD use `0x07ED` only with a trigger a user has confirmed against the device, and MUST use `0x001B` or `0x00A2` for amplifier zones.
- **Request Payload (19 bytes, `length = 21`):** The controller does not check the payload length. A shorter payload is processed and echoed with the missing fields read as zero (§5.4) `[CONFIRMED]`. Clients MUST send all 19 bytes.

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 1 | `prefix` | uint8 | MUST be `0x00` (§2.1.1) `[CONFIRMED]` |
| 0x01 | 4 | `trigger_code` | uint32 LE | `(object_id << 16) | button_id`, with the same meaning as in §3.1. `object_id` is a project user-interface object identifier, not an `mcs.dat` device identifier `[STATIC]`. Clients MUST NOT set bit 12 (`0x1000`) of `button_id`: when it is set, the reply body no longer follows the layout below `[CONFIRMED]`. |
| 0x05 | 4 | `min_value` | uint32 LE | Minimum level bound `[STATIC]` |
| 0x09 | 4 | `max_value` | uint32 LE | Maximum level bound `[STATIC]` |
| 0x0D | 4 | `current_value` | uint32 LE | Requested level `[STATIC]` |
| 0x11 | 1 | `mute_status` | uint8 | `0x00` = unmuted, `0x01` = muted `[STATIC]` |
| 0x12 | 1 | `reserved` | uint8 | Not echoed; MUST be sent as `00` `[CONFIRMED]` |

- **Response Opcode:** `0x07EE` (not `0x07DB`) `[CONFIRMED]`.
- **Response Frame (23 bytes):** `00 14 00 EE 07 01` followed by a 17-byte body.

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `trigger_code` | uint32 LE | Copied from the request |
| 0x04 | 4 | `min_value` | uint32 LE | Copied from the request |
| 0x08 | 4 | `max_value` | uint32 LE | Copied from the request |
| 0x0C | 4 | `current_value` | uint32 LE | Copied from the request |
| 0x10 | 1 | `mute_status` | uint8 | Copied from the request |

The body is request payload offsets `0x01`–`0x11` verbatim: `prefix` and `reserved` are not echoed `[CONFIRMED]`. It does not report the resulting device level, and it is returned even for a trigger code that does not exist in the project `[CONFIRMED]`. A body equal to the request's fields therefore confirms framing only. The controller runs the device action before it sends the echo and releases the client address registration just before it (§4 rule 10) `[STATIC]`. `0x07ED` is terminal: the controller closes the connection 10 s after the echo (§4) `[CONFIRMED]`.

### 3.3 Room Power Off (`0x000D`)

- **Name:** `ROOM_POWER_OFF`
- **Request Opcode:** `0x000D`
- **Request Payload (4 bytes):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `room_id` | uint32 LE | Room identifier to power off |

- **Response:** `00 03 00 0F 00 01` (`resp_opcode = 0x000F`, no body) `[CONFIRMED]`. The same frame is returned for an unknown `room_id`, so it does not confirm that a room was found `[CONFIRMED]`.
- **Connection Effect:** Terminal (§4) `[STATIC]`.

### 3.4 House Off (`0x000E`)
- **Name:** `HOUSE_OFF`
- **Request Opcode:** `0x000E`
- **Request Payload:** None (`length = 2`)
- **Effect:** Runs the project's system-off macro (`32-download-database` §5.6) `[CONFIRMED]`. In the reference installation it switched every amplifier zone off and ran each room's power-off steps.
- **Response** `[CONFIRMED]`:
  1. `00 05 00 DB 07 01 01 02` (`0x07DB`), sent at once. Its first byte is `0x00`, unlike the `0x07DB` sent for `0x07DA`; clients MUST NOT rely on it (§2.2).
  2. Zero or more `0x07DD` frames (`00 04 00 DD 07 01 01` and `00 04 00 DD 07 01 00`) while the macro steps run. The reference produced six pairs.
  3. `00 03 00 0F 00 01` (`resp_opcode = 0x000F`, no body) when the macro has finished: 3.77 s after the request in the reference. The time depends on the project's macro.
- **Connection Effect:** Terminal (§4 rule 3), like `0x000D`. After `0x000F` the controller waits up to 10,000 ms and closes the connection; it closed 10.0 s after `0x000F` `[CONFIRMED]`. A frame sent during that wait gets no response and ends the wait `[STATIC]`. Clients MUST read until `0x000F` and then close.

### 3.5 Power Menu Query (`0x0012`)
- **Name:** `POWER_MENU_QUERY`
- **Request Opcode:** `0x0012`
- **Description:** Read-only query that returns the controller's global power-menu list followed by the power-menu list of one room `[STATIC]`. It switches nothing on: no network-observable device received a command when it was sent for any room of the reference installation `[CONFIRMED]`. Earlier revisions named it room power-on; that was wrong. No room power-on opcode is defined in this specification; in the reference installation a room was switched on by triggering one of its activity buttons with `0x07DA` (§3.1) `[CONFIRMED]`.
- **Request Payload (4 bytes, `length = 6`):** `room_id` (uint32 LE). A 5-byte form with a leading `0x00` prefix byte was answered identically. Because every list in the reference installation was empty, the offset from which the controller reads `room_id` is not established.
- **Response** `[CONFIRMED]`: `00 0B 00 14 00 01` (`resp_opcode = 0x0014`, status `0x01`) followed by this body:

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `global_count` | uint32 LE | Number of entries in the global power-menu list. Observed `0` `[CONFIRMED]` |
| 0x04 | variable | `global_entries` | bytes | `global_count` entries; entry layout not established `[STATIC]` |
| `0x04 + size(global_entries)` | 4 | `room_count` | uint32 LE | Number of entries in the room's power-menu list. Observed `0` `[CONFIRMED]` |
| `0x08 + size(global_entries)` | variable | `room_entries` | bytes | `room_count` entries; entry layout not established `[STATIC]` |

With both counts zero the body is 8 bytes (`resp_length = 0x000B`). Rooms 0, 1, and 2 of the reference installation returned the same reply, so the reply alone does not show which room was read `[CONFIRMED]`. An earlier revision observed a 4-byte body (`resp_length = 0x0007`) for an unknown room; whether an unknown room shortens the reply is not established. Clients MUST bound the body by `resp_length` and MUST NOT parse entries until their layout is specified.
- **Connection Effect:** Terminal (§4 rule 3): the controller closes the connection 10 s after the reply `[CONFIRMED]`.

### 3.6 System Variable Table Query (`0x089A`)
- **Name:** `VARIABLE_TABLE_QUERY`
- **Request Opcode:** `0x089A`
- **Request Payload:** None (`length = 2`). A `prefix` byte `0x00` (`length = 3`) is accepted and gives the same reply `[CONFIRMED]`.
- **Response Opcode:** `0x0898`, `status = 0x01`
- **Response Payload ($4 + 5 \times k$ bytes):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `count` | uint32 LE | Number of Boolean variables the project declares (see below) `[CONFIRMED]` |
| 0x04 | $5 \times k$ | `entries` | record[$k$] | 5-byte variable status records `[CONFIRMED]` |

Each 5-byte variable status entry consists of:

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| `+0x00` | 4 | `var_id` | uint32 LE | Variable ID (the `n` of MQTT `variable_id_{n}`) `[CONFIRMED]` |
| `+0x04` | 1 | `value` | uint8 | Boolean state `[CONFIRMED]` |

- **Boolean variables only:** the reply lists the project's Boolean (type `0`, `32-download-database` §5.0.2) variables, in the project's variable-table order. Integer and String variables never appear `[STATIC]`. A `var_id` listed here is therefore Boolean; the reply carries no type field.
- **Count:** `count` is taken from the project's Boolean-variable list (`32-download-database` §5.0.3), and the controller stops after `count` records or at the end of its Boolean variables, whichever comes first, so $k \le$ `count` `[STATIC]`. In a correctly generated project the two agree. Clients MUST bound parsing by the frame length, not by `count` alone.
- **Value:** `0` or `1` once the variable has been set at runtime `[CONFIRMED]`. Before the first runtime set it is the first byte of the variable's initial value as stored in the project, which the controller does not range-check `[STATIC]`. Clients SHOULD treat any non-zero value as true.
- **Example (no Boolean variables):** `02 02 00 9A 08` → `00 07 00 98 08 01 00 00 00 00` `[CONFIRMED]`.

#### 3.6.1 Variable Values Query (`0x089B`)

- **Name:** `VARIABLE_VALUES_QUERY`
- **Request Payload (`length = 7 + 4 × count`):**

| Offset | Size | Field | Type | Description | Confidence |
|---|---|---|---|---|---|
| 0x00 | 1 | `prefix` | uint8 | `0x00` (§2.1.1). Without it, the controller reads `count` one byte late | `[CONFIRMED]` |
| 0x01 | 4 | `count` | uint32 LE | Number of IDs that follow | `[CONFIRMED]` |
| 0x05 | 4 × `count` | `var_ids` | uint32 LE array | Variable IDs to read | `[CONFIRMED]` |

  The controller does not check `count` against the frame length; clients MUST send exactly `count` IDs `[STATIC]`.
- **Response:** `status = 0x01`. Body: `count` echoed (uint32 LE), then one 5-byte record (`var_id` uint32 LE, `value` uint8, as in §3.6) per requested ID, in request order `[STATIC]`.
- **Response opcode:** `0x0898` when every requested ID names a Boolean variable `[STATIC]`. Otherwise `0x0899`: the controller stops at the first ID that is unknown **or not Boolean**, and the body holds only the records for the IDs before it `[CONFIRMED]` (unknown ID) / `[STATIC]` (not Boolean). Clients MUST NOT read `0x0899` as "unknown ID" alone.
- **Connection Effect:** Terminal (§4 rule 3) `[CONFIRMED]`.
- **Examples:**

| Request | Reply | Meaning |
|---|---|---|
| `02 07 00 9B 08 00 00 00 00 00` | `00 07 00 98 08 01 00 00 00 00` | `count = 0` `[CONFIRMED]` |
| `02 0B 00 9B 08 00 01 00 00 00 01 00 00 00` | `00 07 00 99 08 01 01 00 00 00` | ID `1` unknown or not Boolean; no records `[CONFIRMED]` |
| `02 0F 00 9B 08 00 02 00 00 00 01 00 00 00 02 00 00 00` | `00 07 00 99 08 01 02 00 00 00` | Stops at the first ID; no records `[CONFIRMED]` |

#### 3.6.2 Boolean Variable Change Notification (UDP `0x089D`)

When a Boolean variable's state changes at runtime, the controller sends an 11-byte UDP datagram to multicast group `239.255.95.91`, port `49950` `[STATIC]`:

| Offset | Size | Field | Value |
|---|---|---|---|
| 0x00 | 3 | header | `00 08 00` (`length = 8`) |
| 0x03 | 2 | `opcode` | `0x089D` (`9D 08`) |
| 0x05 | 1 | `status` | `0x01` |
| 0x06 | 4 | `var_id` | uint32 LE |
| 0x0A | 1 | `value` | `0` or `1` |

Each change is sent twice, about 10 ms apart. Nothing is sent at startup, when a set leaves the state unchanged, or for Integer and String variables `[STATIC]`. Opcode `0x089C` is never sent `[STATIC]`. Clients MAY join the group and MUST tolerate duplicates. No datagram has been captured live.

#### 3.6.3 Reading Variables: Summary

| Variable type | Type known from | Current value readable from |
|---|---|---|
| Boolean (`0`) | Listed by `0x089A`; `0x0898` reply to `0x089B`; MQTT `variable_type` | `0x089A`, `0x089B`, `0x089D`, MQTT `variable_status` (`00-system-overview` §5.2.2 item 3) |
| Integer (`1`) | MQTT `variable_type` only | Not readable. MQTT carries only the low byte of the initial value, published at controller start; runtime changes are reported on no channel `[STATIC]` |
| String (`2`) | MQTT `variable_type` only | Initial value only, from MQTT at controller start; runtime changes are reported on no channel `[STATIC]` |

No TCP 51510 request and no staged client file distinguishes an Integer from a String variable `[STATIC]`. When the retained MQTT message is missing (`00-system-overview` §5.2.2 item 3), a client can only tell Boolean (listed by `0x089A`) from not Boolean or unknown.

### 3.7 Network Status Query (`0x0FE1`)
- **Name:** `NETWORK_STATUS_QUERY`
- **Request Opcode:** `0x0FE1`
- **Request Payload:** None (`length = 2`)
- **Response:** `00 03 00 FD FF 01` (online) or `00 03 00 FE FF 01` (not online). No body. The status byte is `0x01` in both cases; the response opcode carries the result `[STATIC]`. The online frame is `[CONFIRMED]`.
- **Status byte:** Every `0xFFFD` reply to this query carries status `0x01`. Status `0x00` is never emitted in a `0xFFFD` reply on this port `[STATIC]`, and none was seen in any reply `[CONFIRMED]`. Clients MUST treat `0xFFFD` as online regardless of its status byte, and SHOULD treat any status other than `0x01` as reserved.
- **Meaning of online:** The controller periodically checks its wired network interface. The reply is `0xFFFD` only when the most recent check found the interface up with an IPv4 address. It is `0xFFFE` when the link is down, when no address could be read, or before the first check completes after startup `[STATIC]`. It does not test reachability of any remote service.
- **Connection Effect:** Terminal (§4) `[CONFIRMED]`.

### 3.8 Macro Resource Type Query (`0x0FE5`)
- **Name:** `RESOURCE_TYPE_QUERY`
- **Request Opcode:** `0x0FE5`
- **Request Payload (10 bytes, `length = 12`):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 2 | `button_id` | uint16 LE | Button index |
| 0x02 | 4 | `device_id` | uint32 LE | Device object identifier |
| 0x06 | 4 | `room_id` | uint32 LE | Room identifier |

- **Response Opcode:** `0xFFFD`
- **Response Payload:** Returns a resource-binding classification (for example IR, serial/network driver, or multi-zone-amplifier volume pass-through) `[CONFIRMED]`. If issued with an empty payload, returns an unpopulated status response (`00 03 00 fd ff 01`).

### 3.9 Controller Time and Time-Zone Query (`0x0FE6`)
- **Name:** `CONTROLLER_TIME_QUERY`
- **Request Opcode:** `0x0FE6`
- **Request Payload:** None (`length = 2`)
- **Response Opcode:** `0xFFFD`
- **Response Body:** 48 bytes (`resp_length = 0x0033`; total wire size 54 bytes):

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| `0x00` | 4 | `local_timestamp` | uint32 LE | Local wall-clock time encoded as seconds since 1970-01-01 00:00:00 (see below) `[CONFIRMED]` |
| `0x04` | 44 | `time_zone_rule` | struct | Configured time-zone rule (layout below). It is not a project identity token `[CONFIRMED]` |

**Timestamp semantics.** The controller's system clock runs in UTC. `local_timestamp = utc_seconds − 60 × (bias + daylight_bias)` while daylight-saving time is in effect, and `utc_seconds − 60 × (bias + standard_bias)` otherwise. The standard-time case is `[CONFIRMED]` (observed value equal to UTC minus `60 × bias` seconds); the daylight-saving case is `[STATIC]`. The value is therefore a local wall-clock reading, not a UTC instant: formatting it as UTC displays the controller's local date and time. To obtain UTC, add back `60 ×` the applicable total offset.

**`time_zone_rule` layout** `[STATIC]`; observed field values matched a configured North American zone `[CONFIRMED]`:

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| `+0x00` | 4 | `bias` | int32 LE | Minutes to add to local standard time to obtain UTC (for example `300` for UTC−5) |
| `+0x04` | 16 | `standard_date` | 8 × uint16 LE | Transition to standard time: `year, month, day_of_week, week, hour, minute, second, millisecond`. With `year = 0`, `week` selects the Nth `day_of_week` (0 = Sunday) of `month` (observed values 1 and 2); `week = 5` meaning the last occurrence is `[INFERRED]` |
| `+0x14` | 4 | `standard_bias` | int32 LE | Additional minutes during standard time (normally `0`) |
| `+0x18` | 16 | `daylight_date` | 8 × uint16 LE | Transition to daylight time, same encoding. `month = 0` means the zone does not observe daylight saving |
| `+0x28` | 4 | `daylight_bias` | int32 LE | Additional minutes during daylight time (normally `-60`) |

`0x0FE6` does not end the connection. A client MAY issue another request after consuming the complete 54-byte response. Repeated `0x0FE6` requests on one connection are supported `[CONFIRMED]`.

Clients MUST NOT use `0x0FE6` to detect project deployments. Project revision detection uses staged model filenames/checksums and project metadata described in Sections 30 and 53.

### 3.10 Client Login (`0x1389`)
- **Name:** `CLIENT_LOGIN_REQUEST`
- **Request Opcode:** `0x1389`
- **Request Payload (13 bytes, total `length = 15`):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `client_id` | uint32 LE | Client seat identifier (matching the Table 4 client table in `DnData.dn`) |
| 0x04 | 9 | `padding` | bytes | Reserved / zero padding bytes |

- **Response Opcode:** `0x138C` (`CLIENT_LOGIN_ACCEPT`) on authorization, or `0x13EB` (`CLIENT_LOGIN_REJECT`) on rejection.
- **Response Payload (5 bytes on ACK, `resp_length = 5`):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 2 | `auth_id` | uint16 LE | Allocated session authorization handle (typically `1`) |

#### Login Behavior
1. **Seat Registration:** Session login (`0x1389`) registers a client seat (see [Client Protocols](53-client-protocols.md) §5). The command opcodes in §3.1–§3.9 and §3.11 carry no session or authorization field and are framed independently of `0x1389` `[CONFIRMED]`.
2. **Concurrency & Re-login:** When `0x1389` is called, the controller tracks client sessions by hardware MAC address (supporting up to 200 active client records). If multiple client devices connect using the same seat ID, each is accepted as long as the seat is registered in Table 4. If an existing client reconnects from the same MAC address, its session is updated and re-acknowledged without disconnecting other clients `[CONFIRMED]`.

### 3.11 Room and Amplifier-Zone Audio Commands

These commands set level, mute, power, or source on a room's audio zone, or directly on one zone of a multi-zone amplifier (HDA family, connected-device sub-type `0x0105`, `32-download-database` §5.3). Each opcode in §3.11.3–§3.11.9 except `0x0144` has been confirmed to change a zone of a live amplifier `[CONFIRMED]`.

#### 3.11.1 Common Rules

- **Framing:** Every payload begins with a 1-byte `prefix` that MUST be `0x00`, as for `0x07DA` (§2.1.1). Fields start at payload offset `0x01` (§2.1.1 gives the confidence per opcode). All multi-byte integers are little-endian. `length` is 2 plus the payload size.
- **Reply timing:** The controller sends its reply only after the amplifier has answered the command it sent: about 10–20 ms after the request for `0x00A2`, `0x001B`, `0x0006`, `0x007E`, `0x0098`, `0x0132`, and `0x012E`, and about 100 ms after the amplifier's reply for `0x0142` and `0x014A` `[CONFIRMED]`. When the amplifier does not answer, the reply follows an amplifier timeout of about 2 s `[CONFIRMED]`. Clients SHOULD allow at least 3 s for the reply.
- **A success reply does not prove the change:** success means that the controller built and sent the amplifier command. It does not prove that the amplifier applied it: §3.11.8 describes a success reply for a command the amplifier ignored `[CONFIRMED]`. Clients that need the resulting state SHOULD read it from MQTT zone telemetry (`00-system-overview` §5.2.2). That telemetry is published for zones bound to a room; a change to a zone bound to no room publishes nothing `[CONFIRMED]`.
- **Lifecycle:** Clients MUST treat these commands as terminal (§4 rules 3 and 4): after its reply the controller waits up to 10,000 ms for another frame `[STATIC]`. Clients MUST close the connection after the reply and use a new connection for the next request. When these opcodes release the client address registration (§4 rule 10) is not established; clients MUST assume it lasts until the connection closes.
- **Replies:**

| Opcodes | Success | Failure | Confidence |
|---|---|---|---|
| `0x00A2`, `0x001B` | `01 03 00 03 00 01` (`resp_opcode = 0x0003`, status `0x01`, no body) | `01 03 00 FE FF 01` | `[CONFIRMED]` success / `[STATIC]` failure |
| `0x0006`, `0x007E`, `0x0098` | `00 03 00 03 00 02` (`resp_opcode = 0x0003`, status `0x02`, no body) | `00 03 00 FE FF 02` | `[CONFIRMED]` |
| `0x0142`, `0x014A`, `0x0132` | `01 03 00 2C 01 01` (`resp_opcode = 0x012C`, no body) | `resp_opcode = 0x012D`; never observed | `[CONFIRMED]` success / `[STATIC]` failure |
| `0x0144` | `01 03 00 2C 01 01` | `resp_opcode = 0x012D` | `[STATIC]` |
| `0x012E` | `resp_opcode = 0x012C`; observed `01 03 00 2C 01 01` | `resp_opcode = 0x012D`; never observed | `[CONFIRMED]` success / `[STATIC]` failure |
| `0x0005`, `0x007D`, `0x0097` | Not established | `00 03 00 FE FF 02` (§3.11.10) | `[CONFIRMED]` failure |
| `0x000A` | No reply | — | `[STATIC]` |

  Clients MUST decide success from `resp_opcode` (`0x0003` or `0x012C` versus `0xFFFE` or `0x012D`), not from `status`.

#### 3.11.2 Zone Identification

The commands name an amplifier zone in one of four ways:

| Name | Range (24-output amplifier) | Used by | Meaning |
|---|---|---|---|
| Room | Room ID (the value used by `0x000D`, §3.3) | `0x00A2`, `0x0006`, `0x007E` | The room's audio routing descriptor (`32-download-database` §5.3, Table 10) names a connected device and a zone-output index `[STATIC]`. Only a room whose descriptor links an amplifier zone reaches one |
| Trigger code | `(object_id << 16) \| button_id` | `0x001B` | Resolved to a binding as for `0x07ED` (§3.2 step 1) `[CONFIRMED]` |
| Zone-output index `zone_index` | `0`–`23` | `0x0098` | 0-based position in the amplifier's zone-output list: the index used by Table 5 zone-output lists, room routing descriptors, and device-command macro steps (`32-download-database` §5.2.1, §5.3). In the reference the list is ordered `z1lr`…`z8lr`, `z1l`…`z8l`, `z1r`…`z8r` `[STATIC]`; index `1` addressed `z2lr` `[CONFIRMED]` |
| Controller slot `slot` | `0`–`15` | `0x0142`, `0x014A` | The slot used by MQTT topic `hda/zone/{hda}/{zone}/` (`00-system-overview` §5.2.2): `2(n−1)` for stereo zone *n* unsplit (`z<n>lr`) or its left half (`z<n>l`), `2(n−1)+1` for its right half (`z<n>r`). Slot `2` addressed `z2lr` `[CONFIRMED]` |
| Zone-ID string | `z<n>lr`, `z<n>l`, `z<n>r` | `0x0132`, `0x012E` | The amplifier's own zone name, carried in its JSON command `[CONFIRMED]` |

The amplifier is named by `hda_id`, its connected-device ID (`mcs.dat` `con_devs[].id`, equal to the Table 5 device ID) `[CONFIRMED]`. That `mcs.dat` `zones[].idx` equals the zone-output index is `[INFERRED]`: both are 0-based lists with one entry per zone output. Where a command accepts it, clients SHOULD name a zone by slot or zone-ID string.

#### 3.11.3 Room Audio (`0x00A2`)

- **Name:** `ROOM_AUDIO_SET`
- **Request Payload (8 bytes, `length = 10`):**

| Offset | Size | Field | Type | Allowed Values | Description | Confidence |
|---|---|---|---|---|---|---|
| 0x00 | 1 | `prefix` | uint8 | `0x00` | §2.1.1 | `[CONFIRMED]` |
| 0x01 | 4 | `room_id` | uint32 LE | Room ID | Room whose audio routing selects the zone | `[CONFIRMED]` |
| 0x05 | 1 | `control` | uint8 | `0`–`3` | `0` absolute level; `1` mute; `2` one step up; `3` one step down | `[CONFIRMED]` |
| 0x06 | 2 | `value` | uint16 LE | See description | `control = 0`: raw zone level. `control = 1`: `0` off, `1` on, `2` toggle. `control = 2` or `3`: ignored; send `0` | `[CONFIRMED]` (level, on, off) / `[STATIC]` (toggle, ignored) |

- **Zone selection:** `room_id` → the room's routing descriptor → connected device and zone-output index → amplifier zone `[CONFIRMED]`.
- **Amplifier command:** `control = 0` sets `"volume":"<level>"`; `2` and `3` send `"volume":"+"` and `"volume":"-"`, which move the zone by the amplifier's own step (2 levels on the reference amplifier) `[CONFIRMED]`; `1` sends `"mute":"on"`, `"off"`, or `"toggle"` `[CONFIRMED]` (on, off) / `[STATIC]` (toggle).
- **Level scale:** `value` is the zone's raw level, not a percentage. A client converting from a percentage SHOULD send `min + (max − min) × percent / 100`, using the zone's configured `min_vol` and `max_vol` (Section 30) `[STATIC]`.
- **Replies:** §3.11.1. Behaviour for room links to device types other than an amplifier is not established.

#### 3.11.4 Level by Trigger (`0x001B`)

- **Name:** `TRIGGER_LEVEL_SET`
- **Request Payload (18 bytes, `length = 20`):**

| Offset | Size | Field | Type | Allowed Values | Description | Confidence |
|---|---|---|---|---|---|---|
| 0x00 | 1 | `prefix` | uint8 | `0x00` | §2.1.1 | `[CONFIRMED]` |
| 0x01 | 4 | `trigger_code` | uint32 LE | Project trigger code | `(object_id << 16) \| button_id`, resolved as in §3.2 step 1 | `[CONFIRMED]` |
| 0x05 | 4 | `min_value` | uint32 LE | Any | Ignored | `[STATIC]` |
| 0x09 | 4 | `max_value` | uint32 LE | Any | Ignored | `[STATIC]` |
| 0x0D | 4 | `level` | uint32 LE | Raw zone level | Requested absolute level; only the low byte is sent to an amplifier | `[CONFIRMED]` |
| 0x11 | 1 | `mute_status` | uint8 | `0x00` | Ignored | `[STATIC]` |

  Unlike `0x07ED`, the payload has no trailing `reserved` byte and the reply is not an echo.
- **Zone selection:** the binding found for `trigger_code`. A room-menu, a device-object, and a module-device volume trigger (`button_id = 275`) each reached the reference amplifier zone `[CONFIRMED]`. Only zones that some project binding reaches can be addressed this way.
- **Device action:** if the bound connected device has sub-type `0x0100`, `0x0103`, `0x0104`, or `0x0105` (`32-download-database` §5.3), the controller sets `level` on the bound zone directly; for an amplifier it sends `"volume":"<level>"` `[CONFIRMED]`. For any other device it performs the `0x07ED` device action (§3.2 step 2) `[STATIC]`.
- **Replies:** §3.11.1. Which conditions produce the failure frame is not established, and a success frame does not show whether the `0x07ED` device action did anything.

#### 3.11.5 Mute (`0x0006`, `0x007E`, `0x0098`)

- **Names:** `ROOM_MUTE_SET` (`0x0006`), `ROOM_MUTE_SET_EXT` (`0x007E`), `ZONE_MUTE_SET` (`0x0098`)
- **Request Payloads:**

| Opcode | Offset | Size | Field | Type | Allowed Values | Description | Confidence |
|---|---|---|---|---|---|---|---|
| `0x0006` (`length = 5`) | 0x00 | 1 | `prefix` | uint8 | `0x00` | §2.1.1 | `[CONFIRMED]` |
| | 0x01 | 1 | `room` | uint8 | Room ID ≤ 255 | Room whose audio routing selects the zone | `[CONFIRMED]` |
| | 0x02 | 1 | `mute` | uint8 | `0`–`2` | `0` off, `1` on, `2` toggle | `[CONFIRMED]` (on) / `[STATIC]` (off, toggle) |
| `0x007E` (`length = 8`) | 0x00 | 1 | `prefix` | uint8 | `0x00` | §2.1.1 | `[CONFIRMED]` |
| | 0x01 | 4 | `room_id` | uint32 LE | Room ID | As for `0x0006` | `[CONFIRMED]` |
| | 0x05 | 1 | `mute` | uint8 | `0`–`2` | As for `0x0006` | `[CONFIRMED]` (on) / `[STATIC]` (off, toggle) |
| `0x0098` (`length = 9`) | 0x00 | 1 | `prefix` | uint8 | `0x00` | §2.1.1 | `[CONFIRMED]` |
| | 0x01 | 4 | `con_dev_id` | uint32 LE | Connected-device ID | The amplifier (`hda_id`) | `[CONFIRMED]` |
| | 0x05 | 1 | `zone_index` | uint8 | `0`–`23` | Zone-output index (§3.11.2) | `[CONFIRMED]` |
| | 0x06 | 1 | `mute` | uint8 | `0`–`2` | As for `0x0006` | `[CONFIRMED]` |

- **Zone selection:** `0x0006` and `0x007E` use the room's routing descriptor, as `0x00A2`. `0x0098` takes the connected device and zone-output index from the request and needs no room binding, so it reaches any zone, including one bound to no room `[CONFIRMED]`. It requires only that the project contain at least one room `[STATIC]`.
- **Amplifier command:** `"mute":"off"`, `"on"`, or `"toggle"` on the selected zone `[CONFIRMED]` (off, on).
- **Replies:** success is `00 03 00 03 00 02`: status `0x02` on success `[CONFIRMED]`. An unknown room is answered `00 03 00 FE FF 02` `[CONFIRMED]`.

#### 3.11.6 Amplifier Zone Power (`0x0142`)

- **Name:** `AMP_ZONE_POWER_SET`
- **Request Payload (`5 + 9 × count` bytes, `length = 7 + 9 × count`):**

| Offset | Size | Field | Type | Allowed Values | Description | Confidence |
|---|---|---|---|---|---|---|
| 0x00 | 1 | `prefix` | uint8 | `0x00` | §2.1.1 | `[CONFIRMED]` |
| 0x01 | 4 | `count` | uint32 LE | ≥ 1 | Number of entries that follow | `[CONFIRMED]` |
| 0x05 + 9i | 4 | `hda_id` | uint32 LE | Connected-device ID | Amplifier of entry *i* | `[CONFIRMED]` |
| 0x09 + 9i | 4 | `slot` | uint32 LE | `0`–`15` | Controller slot (§3.11.2) | `[CONFIRMED]` |
| 0x0D + 9i | 1 | `power` | uint8 | `0`, other | `0` off; any other value on | `[CONFIRMED]` (`0`, `1`) |

- **Amplifier command:** `"power":"on"` or `"power":"off"` on each entry's zone `[CONFIRMED]`.
- **Replies:** `01 03 00 2C 01 01`, about 100 ms after the amplifier's reply `[CONFIRMED]`.

#### 3.11.7 Amplifier Multi-Zone Mute (`0x014A`) and All-Zone Mute (`0x0144`)

- **Names:** `AMP_ZONES_MUTE_SET` (`0x014A`), `AMP_ALL_MUTE_SET` (`0x0144`)
- **`0x014A` Request Payload (`6 + Σ (8 + 4 × nₕ)` bytes):**

| Offset | Size | Field | Type | Allowed Values | Description | Confidence |
|---|---|---|---|---|---|---|
| 0x00 | 1 | `prefix` | uint8 | `0x00` | §2.1.1 | `[CONFIRMED]` |
| 0x01 | 1 | `mute` | uint8 | `0`, other | `0` off; any other value on | `[CONFIRMED]` (`0`, `1`) |
| 0x02 | 4 | `hda_count` | uint32 LE | ≥ 1 | Number of amplifier groups that follow | `[CONFIRMED]` |
| group + 0x00 | 4 | `hda_id` | uint32 LE | Connected-device ID | Amplifier of this group | `[CONFIRMED]` |
| group + 0x04 | 4 | `n` | uint32 LE | ≥ 1 | Number of slots in this group | `[CONFIRMED]` |
| group + 0x08 | 4 × `n` | `slots` | uint32 LE array | `0`–`15` each | Controller slots (§3.11.2) | `[CONFIRMED]` |

  Groups follow one another from payload offset `0x06`.
- **`0x0144` Request Payload (2 bytes, `length = 4`):** `prefix` (uint8, `0x00`), `mute` (uint8: `0` off, `1` on, `2` toggle). It applies to every zone of the project's master amplifier (`"id":"all"`) `[STATIC]`; not exercised live.
- **Amplifier command:** `"mute":"on"` or `"mute":"off"` on each listed zone `[CONFIRMED]`.
- **Replies:** `01 03 00 2C 01 01`, about 100 ms after the amplifier's reply `[CONFIRMED]`.

#### 3.11.8 Amplifier Zone Source (`0x0132`)

- **Name:** `AMP_ZONE_SOURCE_SET`
- **Request Payload (`10 + 36 × count` bytes, `length = 12 + 36 × count`):**

| Offset | Size | Field | Type | Allowed Values | Description | Confidence |
|---|---|---|---|---|---|---|
| 0x00 | 1 | `prefix` | uint8 | `0x00` | §2.1.1 | `[CONFIRMED]` |
| 0x01 | 4 | `count` | uint32 LE | ≥ 1 | Number of target zones that follow | `[CONFIRMED]` |
| 0x05 | 1 | `input_port` | uint8 | 1-based | Input of the source amplifier to route | `[CONFIRMED]` |
| 0x06 | 4 | `source_hda_id` | uint32 LE | Connected-device ID | Amplifier that owns `input_port` | `[CONFIRMED]` |
| 0x0A + 36i | 4 | `hda_id` | uint32 LE | Connected-device ID | Amplifier of target *i* | `[CONFIRMED]` |
| 0x0E + 36i | 32 | `zone_id` | char[32] | ASCII, NUL-padded | Zone-ID string of target *i* (§3.11.2) | `[CONFIRMED]` |

- **Amplifier command:** the zone is switched on and routed to the input in one command: `"input":"<input name>.<source MAC>","power":"on"`, where `<source MAC>` is the source amplifier's MAC address and `<input name>` is lower case, for example `in1` `[CONFIRMED]`.
- **Unusable input ports:** for an input port that has no source device bound in the project, the controller sends the amplifier an empty input name. The amplifier does not answer and leaves the zone unchanged, and the controller still returns `01 03 00 2C 01 01` after about 2 s `[CONFIRMED]`. The input name is set only for inputs that have a bound source `[STATIC]`. An `input_port` beyond the amplifier's input list, or an unknown `source_hda_id`, produces the failure reply `[STATIC]`.
- **Finding the bound ports:** `0x0130` (§3.11.12) lists every bound input of every amplifier `[CONFIRMED]`. Clients MUST send `0x0132` only with a (`source_hda_id`, `input_port`) pair that appears in a `0x0130` reply read since the last project change, and SHOULD verify the result from zone telemetry. MQTT `hda/input` topics exist for unbound inputs too, with the same body, and MUST NOT be used to decide whether an input is bound `[CONFIRMED]`.
- **Replies:** §3.11.1.

#### 3.11.9 Amplifier JSON Pass-Through (`0x012E`)

- **Name:** `AMP_JSON_COMMAND`
- **Request Payload (`7 + json_length` bytes):**

| Offset | Size | Field | Type | Allowed Values | Description | Confidence |
|---|---|---|---|---|---|---|
| 0x00 | 1 | `prefix` | uint8 | `0x00` | §2.1.1 | `[CONFIRMED]` |
| 0x01 | 4 | `hda_id` | uint32 LE | Connected-device ID, or `0xFFFFFFFF` | Target amplifier; `0xFFFFFFFF` addresses every amplifier | `[CONFIRMED]` (ID) / `[STATIC]` (`0xFFFFFFFF`) |
| 0x05 | 2 | `json_length` | uint16 LE | ≥ 1 | Length of `json` in bytes | `[CONFIRMED]` |
| 0x07 | `json_length` | `json` | bytes | UTF-8 JSON object, no terminator | Amplifier command | `[CONFIRMED]` |

- **Processing:** the controller parses the JSON, checks its zone `id` key and `ctrl` format, and re-serialises it before sending, so key order can change (for example `{"type":"set","zone":{"ctrl":{"mute":"on"},"id":"z2lr"}}`) `[CONFIRMED]`. Clients MUST NOT rely on byte-exact forwarding.
- **Command forms:** the amplifier's JSON grammar is outside this specification. The forms the controller itself sends are `{"type":"set","zone":{"id":"<zone id>","ctrl":{...}}}` with `ctrl` members `"volume"` (`"<level>"`, `"+"`, `"-"`), `"mute"` (`"on"`, `"off"`, `"toggle"`), `"power"` (`"on"`, `"off"`), and `"input"` (`"<input name>.<source MAC>"`) `[CONFIRMED]`. Pass-through was confirmed with `"mute"`; it is the only command here that can set an absolute level on a zone that no room or trigger binding reaches `[INFERRED]`.
- **Replies:** `resp_opcode = 0x012C`, about 10 ms after the request. Its body length is set by the amplifier send routine; only an empty body (`01 03 00 2C 01 01`) has been observed, and the body's content is not established.

#### 3.11.10 Commands That Do Not Reach an Amplifier Zone

| Opcodes | Payload | Behaviour | Confidence |
|---|---|---|---|
| `0x0005`, `0x007D`, `0x0097` (set level) | `prefix`, then `room` (uint8) / `room_id` (uint32 LE) / `con_dev_id` (uint32 LE) + `zone_index` (uint8), then `value` (uint8) | Supported only for connected devices of sub-type `0x0100` or `0x0103`. For any other device, including an amplifier, nothing is sent and the reply is `00 03 00 FE FF 02` | `[STATIC]`; `00 03 00 FE FF 02` for the room linked to the reference amplifier `[CONFIRMED]` |
| `0x000A` (zone power) | Not established | Accepted and ignored: no reply, nothing sent. Clients SHOULD apply a response timeout and close | `[STATIC]` |
| `0x0004`/`0x007C`/`0x0096` (get), `0x0007`/`0x007F`/`0x0099` (balance), `0x0008`/`0x0080`/`0x009A` (treble), `0x0009`/`0x0081`/`0x009B` (bass), `0x0011` (multi-room level) | Not established | Not established | — |

Clients MUST NOT use these opcodes for amplifier zones.

#### 3.11.11 Choosing a Command for One Amplifier Zone

| Goal | Command |
|---|---|
| Absolute level | `0x00A2` with `control = 0` when the zone is a room's audio zone; `0x001B` when a project trigger binds it; otherwise `0x012E` with `"volume"` |
| One step up or down | `0x00A2` with `control = 2` or `3`; otherwise `0x012E` with `"volume":"+"` or `"-"` |
| Mute | `0x0098` by zone-output index, or `0x014A` by slot; `0x00A2`, `0x0006`, or `0x007E` for a room's audio zone |
| Power | `0x0142` |
| Source | `0x0132`, with an input listed by `0x0130` |

#### 3.11.12 Amplifier Input List (`0x0130`)

- **Name:** `AMP_INPUT_LIST_QUERY`
- **Request Payload:** none (`length = 2`). The controller ignores any payload bytes and needs no `prefix` byte `[CONFIRMED]`.
- **Response:** `resp_opcode = 0x0131`, `status = 0x01`, body below `[CONFIRMED]`. The failure opcode `0x012D` is not caused by any request content `[STATIC]`.

| Offset | Size | Field | Type | Description | Confidence |
|---|---|---|---|---|---|
| 0x00 | 4 | `count` | uint32 LE | Number of records that follow | `[CONFIRMED]` |
| 0x04 + 69i | 4 | `hda_id` | uint32 LE | Connected-device ID of the amplifier that owns the input | `[CONFIRMED]` |
| 0x08 + 69i | 1 | `input_port` | uint8 | 1-based input position on that amplifier, in the numbering used by `0x0132` `input_port` | `[CONFIRMED]` (analog) / `[STATIC]` (built-in) |
| 0x09 + 69i | 64 | `name` | char[64] | Name of the source device bound to the input, NUL-padded. Empty when the name is 64 bytes or longer | `[CONFIRMED]` |

- **Content:** one record for each input, on every amplifier in the project, that has a source device bound in the project; an unbound input has no record `[CONFIRMED]`. Records follow project order: amplifier by amplifier, then input by input `[STATIC]`. A built-in (microphone) input is listed with the microphone's name `[STATIC]`.
- **Lifecycle:** as §3.11.1. Clients SHOULD re-read the list after a project change.

#### 3.11.13 Amplifier Input Detail (`0x013B`)

- **Name:** `AMP_INPUT_DETAIL_QUERY`
- **Request Payload:** none (`length = 2`) `[CONFIRMED]`.
- **Response:** `resp_opcode = 0x013C`, `status = 0x01`. The body is `count` (uint32 LE) followed by `count` records of 142 bytes `[CONFIRMED]`:

| Offset | Size | Field | Type | Description | Confidence |
|---|---|---|---|---|---|
| 0x00 | 32 | `source_name` | char[32] | Name of the bound source device | `[CONFIRMED]` |
| 0x20 | 32 | `amp_name` | char[32] | Amplifier name | `[CONFIRMED]` |
| 0x40 | 32 | `port_label` | char[32] | `Input <n>` for an analog input; `Balanced Input Left` / `Balanced Input Right` for built-in inputs | `[CONFIRMED]` (analog) / `[STATIC]` (built-in) |
| 0x60 | 6 | `amp_mac` | uint8[6] | Amplifier MAC address | `[CONFIRMED]` |
| 0x66 | 1 | `builtin` | uint8 | `0` analog input, `1` built-in input | `[CONFIRMED]` (`0`) / `[STATIC]` (`1`) |
| 0x67 | 1 | `port` | uint8 | 1-based position within its kind (analog or built-in) | `[CONFIRMED]` (analog) |
| 0x68 | 4 | `hda_id` | uint32 LE | Amplifier connected-device ID | `[CONFIRMED]` |
| 0x6C | 32 | `input_type` | char[32] | Input type text, for example `Unbalanced Analog RCA` | `[CONFIRMED]` |
| 0x8C | 1 | `stereo` | uint8 | Stereo mode; `1` observed, other values not established | `[CONFIRMED]` (`1`) |
| 0x8D | 1 | `sensing` | uint8 | Sensing option; `0` observed, other values not established | `[CONFIRMED]` (`0`) |

- **Content:** analog inputs without a bound source are omitted; built-in inputs are always listed `[STATIC]`. String fields are NUL-padded and MAY fill all 32 bytes with no terminator; longer names are truncated `[STATIC]`. Text that the controller learns from the amplifier (such as `input_type`) MAY be empty while the amplifier is offline `[INFERRED]`.
- **Port numbering:** `port` counts within its kind, so it is not the `0x0132` `input_port` for a built-in input or for an amplifier that has built-in inputs. Clients MUST take `0x0132` port numbers from `0x0130` `[STATIC]`.
- **Lifecycle:** as §3.11.1.

---

## 4. Connection Lifecycle & Error Handling

1. **First Request Deadline:** The controller waits up to 10,000 ms after accepting a connection for the first request. If none arrives, it sends `01 03 00 FE FF 01` and closes the connection about 3 s later `[CONFIRMED]`.
2. **One Request at a Time:** Clients MUST consume every response frame for a request (§2.2) before issuing a subsequent request on a reusable connection.
3. **Terminal Commands:** After its final response frame to `MACRO_TRIGGER` (`0x07DA`), `SET_LEVEL_ABSOLUTE` (`0x07ED`), `HOUSE_OFF` (`0x000E`), `POWER_MENU_QUERY` (`0x0012`), `NETWORK_STATUS_QUERY` (`0x0FE1`), `VARIABLE_TABLE_QUERY` (`0x089A`), or `VARIABLE_VALUES_QUERY` (`0x089B`), the controller waits up to 10,000 ms and then closes the socket `[CONFIRMED]`. `ROOM_POWER_OFF` (`0x000D`) and the §3.11.3–§3.11.9 commands follow the same path `[STATIC]`. If the client sends another frame during that wait, those bytes are consumed and discarded, and the controller closes the connection without a response `[CONFIRMED]`. Clients MUST open a new connection for the next request, and SHOULD close their side as soon as the final response frame is read rather than waiting for the controller.
4. **Portable Client Rule:** Unless an opcode is explicitly documented here as reusable, clients MUST treat each request/response transaction as terminal and reconnect for the next request.
5. **Reusable Query and Idle Connections:** `CONTROLLER_TIME_QUERY` (`0x0FE6`) returns without closing the socket. Further requests MAY follow after its complete response `[CONFIRMED]`. There is no idle limit after it: a connection left idle for 1801 s after `0x0FE6` remained open and answered another `0x0FE6`, and the controller sent nothing during the idle time `[CONFIRMED]`. The controller sets no TCP keepalive on port 51510 sockets, so it never detects a peer that disappears without FIN or RST; such a connection, and its address registration (rule 10), persist `[STATIC]`. The only idle-close rule on this port (about 60 s) applies to a programmer version opcode outside this specification `[STATIC]`. Because an open connection blocks every new connection from the same address (rule 10), clients SHOULD close idle connections themselves as soon as they no longer need them, and MUST close a `0x0FE6` connection before issuing a request on another connection.
6. **Accept Serialization and Capacity:** The controller accepts one connection at a time. It reads no request from the next connection until the current one has sent its first request, or has reached the first-request deadline (rule 1) and been closed, about 13 s after it connected `[CONFIRMED]`. Other clients' TCP handshakes complete in the meantime (listen backlog of 64), but their requests wait. A rejected connection (rule 10) whose client closes at once does not delay others `[CONFIRMED]`. At most 64 distinct client addresses can be registered at once; a first request from a further address is answered as in rule 10 `[STATIC]`. Connecting a new client does not disconnect another client `[CONFIRMED]`. Clients MUST send their request immediately after connecting.
7. **Startup:** If the controller has not finished system initialization, or is finishing a project download, it answers an accepted connection with `00 03 00 FE FF 01` before reading any request, and closes it about 0.3 s later `[STATIC]`. Clients SHOULD retry after a delay.
8. **Invalid Opcode:** An unrecognized opcode receives no response, and the connection stays open; a subsequent `0x0FE6` on the same connection is answered `[CONFIRMED]`. The client address stays registered until the connection closes (rule 10). Clients SHOULD apply a response timeout of a few seconds and then close the connection.
9. **Invalid Room, Object, or Trigger:** No uniform error reply exists. `0x07DA` acknowledges unknown trigger codes with `0x07DB` followed by `0x07DC` (§3.1.2); `0x07ED` echoes them normally; `0x000D` returns its normal frame for unknown rooms; the §3.11 mute and level commands return `00 03 00 FE FF 02` for an unknown room `[CONFIRMED]`; `0x00A2` and `0x001B` return `01 03 00 FE FF 01` on failure `[STATIC]`. The `0x0012` reply for an unknown room is not established (§3.5). Clients MUST validate identifiers against the project model before sending.
10. **One Registered Connection per Client Address:** When a connection's first request arrives, the controller registers the connection under its client IPv4 address; source ports are not considered `[STATIC]`. While another connection from the same address is registered, a new connection's first request is not executed: the controller answers `01 03 00 FE FF 01` immediately and does not perform the request `[CONFIRMED]`, then waits up to 10 s for the client to close, and closes `[STATIC]`. The rejected request is not queued `[CONFIRMED]`. A registration lasts:

    | Opcode | Registered until | Confidence |
    |---|---|---|
    | `0x07DA` | The action has finished, just before `0x07DC` | `[CONFIRMED]`: rejections ended at `0x07DC`, 1.31–1.47 s after a six-send volume press |
    | `0x07ED` | Just before `0x07EE` | `[STATIC]` |
    | `0x000D`, `0x000E` | Just before `0x000F` | `[STATIC]` |
    | `0x0FE1`, `0x089A` | Just before the reply | `[STATIC]` |
    | `0x0FE6`, unrecognized opcodes | The connection closes | `[CONFIRMED]` (`0x0FE6`) / `[STATIC]` (unrecognized) |
    | Other opcodes in this specification (`0x0012`, `0x089B`, `0x0FE5`, `0x1389`, §3.11) | Not established; clients MUST assume until the connection closes | — |

    Closing a connection early does not cancel its action or release the address: after a client closed right after `0x07DB`, requests from its address were rejected until the action ended `[CONFIRMED]`. A rejection window therefore has no fixed length; it lasts for the remaining run time of the other connection's action, or for as long as another connection from the address stays open. Connections from other addresses are not affected `[STATIC]`; this has not been tested with a second client host. Clients behind one NAT address share one registration `[INFERRED]`.
11. **Client Connection Strategy:**
    - Clients MUST NOT have more than one connection to the controller open from one host address at a time.
    - For an action, clients MUST read until its final frame (`0x07DC`, `0x07EE`, `0x000F`; the reply for other requests) and then close. The next request MAY follow immediately: a request sent 45 ms after closing at `0x07DC` was accepted `[CONFIRMED]`.
    - Clients MUST close a `0x0FE6` connection before issuing any other request (rule 5).
    - Clients MUST send the request immediately after connecting (rule 6).
    - On an immediate `01 03 00 FE FF 01`, the request was not performed. The client MUST close the connection, MAY retry with backoff starting at 250 ms (for example doubling up to 2 s), and SHOULD give up after about 15 s.
    - Clients MUST NOT close an action connection early and reconnect to repeat or cancel it (rule 10).

---

## 5. Test Vectors

### 5.1 Status Query (`NETWORK_STATUS_QUERY`)

Request bytes (5 bytes):
```
02 02 00 e1 0f
```
Expected response when online (6 bytes):
```
00 03 00 fd ff 01
```
Response when not online (6 bytes):
```
00 03 00 fe ff 01
```

### 5.2 Macro Action (`MACRO_TRIGGER`, Trigger Code 0)

Request bytes (10 bytes: `length = 7`, `prefix = 0x00`, `trigger_code = 0`):
```
02 07 00 da 07 00 00 00 00 00
```
Expected response (14 bytes, two frames):
```
01 05 00 db 07 01 01 02
00 03 00 dc 07 01
```
The two `0x07DB` body bytes are controller state; the observed value is shown. Trigger code `0` resolves to no action, so `0x07DC` follows directly. A trigger that runs an action produces the longer sequence in §3.1.2, for example `00 04 00 dd 07 01 01`, `0x07E1` reports, `00 04 00 dd 07 01 00`, then `00 03 00 dc 07 01`. The controller closes the connection after `0x07DC` (§4 rule 3).

### 5.3 Absolute Volume (`SET_LEVEL_ABSOLUTE`, Level 45, Unmuted)

Request bytes (24 bytes on wire: frame type `0x02`, length `0x0015` = 21, opcode `0x07ED`, payload 19 bytes):
```
02 15 00 ed 07 00 13 01 02 00 00 00 00 00 64 00 00 00 2d 00 00 00 00 00
```
*(Fields: prefix = `00`, trigger_code = `0x00020113` (object 2, button 275), min = `0`, max = `100`, current = `45`, mute = `0`, reserved = `00`).*

Expected response (23 bytes):
```
00 14 00 ee 07 01 13 01 02 00 00 00 00 00 64 00 00 00 2d 00 00 00 00
```
The 17-byte body echoes request payload offsets `0x01`–`0x11`.

### 5.3.1 Missing Prefix

A `MACRO_TRIGGER` request that omits `prefix` for trigger code `0x000D0120` (button 288) sends `20 01 0D 00` as its payload. The controller reads `0x20` as `prefix` (§2.1.1) and replies with a frame beginning `00 33 00 72 17 22`, then closes the connection. The button does not run.

### 5.4 Short Volume Request (No Length Check)

Request with a 17-byte payload (`length = 19`; `mute_status` and `reserved` omitted):
```
02 13 00 ed 07 00 13 01 02 00 00 00 00 00 64 00 00 00 2d 00 00 00
```
Response (23 bytes): the full-length echo, with the missing `mute_status` read as `00`:
```
00 14 00 ee 07 01 13 01 02 00 00 00 00 00 64 00 00 00 2d 00 00 00 00
```
Earlier revisions expected `01 03 00 fe ff 01` here; that frame is the address rejection of §4 rule 10, not a length check. Clients MUST still send the full 19-byte payload.

### 5.5 Fragmented TCP Read (Split Frame Handling)

When receiving framed responses over stream sockets, clients MUST handle frames split across multiple `recv()` calls.

**Scenario:** Response to `NETWORK_STATUS_QUERY` (total 6 bytes `00 03 00 fd ff 01`) arriving in two segments:

Read 1 (3 bytes - Header):
```
00 03 00
```
*Client buffer accumulates 3 bytes; inspects `resp_length = 0x0003`; calculates total expected size = $3 + 3 = 6$ bytes; awaits remaining 3 bytes.*

Read 2 (3 bytes - Payload & Status):
```
fd ff 01
```
*Client buffer reaches 6 bytes; frame is dispatched to opcode handler `0xFFFD` with status `0x01`.*

### 5.6 Terminal Query Sequence

After this request and response:

```text
client -> controller: 02 02 00 E1 0F
controller -> client: 00 03 00 FD FF 01
```

the client MUST close or discard that TCP connection. Sending `02 02 00 E6 0F` on the same connection receives no reply because `NETWORK_STATUS_QUERY` is terminal. The client MUST reconnect before issuing `CONTROLLER_TIME_QUERY`.

### 5.7 `CONTROLLER_TIME_QUERY` Decode

Constructed response for a zone at UTC−5 with North American daylight-saving rules, during standard time, at 2026-11-17 20:00:00 UTC:

```
00 33 00 FD FF 01 F0 6B FC 6A 2C 01 00 00 00 00
0B 00 00 00 01 00 02 00 00 00 00 00 00 00 00 00
00 00 00 00 03 00 00 00 02 00 02 00 00 00 00 00
00 00 C4 FF FF FF
```

- `local_timestamp = 0x6AFC6BF0`; formatted as UTC this reads 2026-11-17 15:00:00, the local wall-clock time.
- `bias = 300`; `standard_date` = month 11, Sunday, week 1, 02:00; `standard_bias = 0`; `daylight_date` = month 3, Sunday, week 2, 02:00; `daylight_bias = -60`.
- UTC = `0x6AFC6BF0 + 60 × (300 + 0)` = 2026-11-17 20:00:00.

### 5.8 First-Request Deadline

A client that connects and sends nothing receives, after 10 s:
```
01 03 00 fe ff 01
```
and the controller then closes the connection.

### 5.9 Room-Addressed Audio Vectors (`0x00A2`, `0x001B`)

Identifiers in §5.9–§5.15 are synthetic: room `4`, whose routing descriptor links amplifier connected-device ID `7` at zone-output index `2`; that zone is `z3lr` (controller slot `4`); trigger code `0x00090113` (object 9, button 275) binds the same zone. Amplifier commands are shown as the controller sends them; key order varies between paths and carries no meaning. The reply layouts and the frame layouts are `[CONFIRMED]`.

Room 4 absolute level 40 (`control = 0`, `value = 0x0028`):
```
02 0a 00 a2 00 00 04 00 00 00 00 28 00
```
Amplifier command: `{"type":"set","zone":{"ctrl":{"volume":"40"},"id":"z3lr"}}`. Reply:
```
01 03 00 03 00 01
```
Room 4 mute on (`control = 1`, `value = 1`), one step up (`control = 2`), and one step down (`control = 3`):
```
02 0a 00 a2 00 00 04 00 00 00 01 01 00
02 0a 00 a2 00 00 04 00 00 00 02 00 00
02 0a 00 a2 00 00 04 00 00 00 03 00 00
```
Each on its own connection; amplifier commands `"mute":"on"`, `"volume":"+"`, `"volume":"-"`; each reply `01 03 00 03 00 01`.

Level 40 by trigger `0x00090113` (`0x001B`, 18-byte payload):
```
02 14 00 1b 00 00 13 01 09 00 00 00 00 00 64 00 00 00 28 00 00 00 00
```
Amplifier command `"volume":"40"` on `z3lr`. Reply `01 03 00 03 00 01`.

### 5.10 Mute One Amplifier Zone (`0x0098`)

Mute on, amplifier `7`, zone-output index `2` (`z3lr`), no room needed:
```
02 09 00 98 00 00 07 00 00 00 02 01
```
Amplifier command: `{"type":"set","zone":{"ctrl":{"mute":"on"},"id":"z3lr"}}`. Reply (status `02` means success here):
```
00 03 00 03 00 02
```
Mute off is the same frame with last byte `00`. Room 4 mute on with `0x0006` and `0x007E`:
```
02 05 00 06 00 00 04 01
02 08 00 7e 00 00 04 00 00 00 01
```
Each reply `00 03 00 03 00 02`.

### 5.11 Amplifier Zone Power (`0x0142`)

Power on, one entry: amplifier `7`, slot `4` (`z3lr`):
```
02 10 00 42 01 00 01 00 00 00 07 00 00 00 04 00 00 00 01
```
Amplifier command: `{"type":"set","zone":{"ctrl":{"power":"on"},"id":"z3lr"}}`. Reply:
```
01 03 00 2c 01 01
```
Power off is the same frame with last byte `00`.

### 5.12 Amplifier Multi-Zone Mute (`0x014A`)

Mute on, amplifier `7`, slots `4` and `6` (`z3lr`, `z4lr`):
```
02 18 00 4a 01 00 01 01 00 00 00 07 00 00 00 02 00 00 00 04 00 00 00 06 00 00 00
```
Amplifier commands: `"mute":"on"` on `z3lr` and `z4lr`. Reply `01 03 00 2c 01 01`.

### 5.13 Amplifier Zone Source (`0x0132`)

Route input 1 of amplifier `7` to `z3lr` of amplifier `7` (`zone_id` = `z3lr` + 28 NUL bytes):
```
02 30 00 32 01 00 01 00 00 00 01 07 00 00 00 07 00 00 00 7a 33 6c 72
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00
```
Amplifier command, for a source amplifier with MAC `00:1F:B8:00:11:22` and a source device bound to input 1: `{"type":"set","zone":{"ctrl":{"input":"in1.00:1f:b8:00:11:22","power":"on"},"id":"z3lr"}}`. Reply `01 03 00 2c 01 01`. The same reply is returned when the input has no bound source and the amplifier ignores the command (§3.11.8).

### 5.14 Amplifier Zone Volume and Mute by JSON (`0x012E`)

Absolute level 40 on `z3lr` of amplifier `7` (`json_length = 0x003A` = 58):
```
02 43 00 2e 01 00 07 00 00 00 3a 00
```
followed by the 58 ASCII bytes `{"type":"set","zone":{"id":"z3lr","ctrl":{"volume":"40"}}}`. The controller re-serialises the object before sending it. Reply `01 03 00 2c 01 01`. This form is `[INFERRED]`; the same frame with `json_length = 0x0038` (`length = 0x0041`) and `{"type":"set","zone":{"id":"z3lr","ctrl":{"mute":"on"}}}` exercises the `[CONFIRMED]` mute form.

### 5.15 House Off and Power Menu Query

House Off (`0x000E`):
```text
client -> controller: 02 02 00 0E 00
controller -> client: 00 05 00 DB 07 01 01 02
controller -> client: 00 04 00 DD 07 01 01      (pairs repeat while the macro runs)
controller -> client: 00 04 00 DD 07 01 00
controller -> client: 00 03 00 0F 00 01
controller closes the connection about 10 s later
```
Power menu query (`0x0012`) for room 4, in a project without power menus:
```text
client -> controller: 02 06 00 12 00 04 00 00 00
controller -> client: 00 0B 00 14 00 01 00 00 00 00 00 00 00 00
controller closes the connection about 10 s later
```

### 5.16 Address Rejection and Retry

From one host (§4 rules 10 and 11), with a volume press on connection A:
```text
A: client -> controller: 02 07 00 DA 07 00 13 01 09 00
A: controller -> client: 01 05 00 DB 07 01 01 02
B: client -> controller: 02 02 00 E1 0F              (sent during A's action)
B: controller -> client: 01 03 00 FE FF 01           (immediately; not performed)
B: client closes; retries on new connections after 250 ms, 500 ms, ...
A: controller -> client: ... 00 03 00 DC 07 01       (action finished; address released)
C: client -> controller: 02 02 00 E1 0F
C: controller -> client: 00 03 00 FD FF 01
```

### 5.17 Amplifier Input List (`0x0130`)

One amplifier (`hda_id = 7`) with a source named `Source` bound to input 1 and no other bound inputs:
```text
client -> controller: 02 02 00 30 01
controller -> client: 01 4C 00 31 01 01
                      01 00 00 00                 count = 1
                      07 00 00 00                 hda_id = 7
                      01                          input_port = 1
                      53 6F 75 72 63 65           "Source"
                      00 × 58                     NUL padding to 64 bytes
```
`resp_length = 0x004C` = 2 (opcode) + 1 (status) + 4 + 69. Inputs 2–8 have no record, so `0x0132` MUST NOT be sent for them.

---

## 6. Open Items

1. **Button-Status Frame `0x07DE`:** The exact layout of `0x07DE`, and which connection property, if any, enables the controller's hold-extension mode. Frame type `0x01` does not (§3.1.4).
2. **`0x07DC` / `0x07DD` Frames:** Whether `0x07DD` bodies other than `01` and `00` exist, and the meaning of the unexplained `0x07E1` bytes (§3.1.3).
3. **Amplifier-Zone Reply Bodies:** The content of a non-empty `0x012C` body after `0x012E`; whether `0x012D` is ever sent; and the bodies of the `0x0003` replies to the room audio state queries `0x00A1` and `0x00A7`, which this specification does not yet define (§3.11).
4. **Power Menu Entries:** The entry layout of the `0x0012` global and room power-menu lists beyond the two counts, the offset from which `room_id` is read, and whether an unknown room shortens the reply (§3.5).
5. **Address Registration:** Rule 10 of §4 has not been tested with a second client host. The release point for `0x0012`, `0x0FE5`, `0x1389`, and the §3.11 commands is not established.
6. **Room Audio Forms:** The get, balance, treble, and bass forms and `0x0011` in §3.11.10; `0x0144` live behaviour; whether `mcs.dat` `zones[].idx` equals the zone-output index (§3.11.2).
7. **`0x07ED` Acceptance:** No device has been observed to accept `0x07ED`; both device paths in §3.2 step 2 are `[STATIC]`.
8. **Controller-to-Client Jump:** When a controller-side macro contains a page jump, the controller pushes `0x07DF` to the client. Its payload layout and delivery path are not established.
9. **Prefix Byte Scope:** Whether opcodes other than `0x07DA`, `0x07ED`, `0x089B`, and §3.11.3–§3.11.9 carry the §2.1.1 prefix. `0x089A`, `0x0130`, and `0x013B` ignore payload bytes.
10. **Amplifier Inputs:** Built-in (microphone) inputs in `0x0130` and `0x013B`, record order across several amplifiers, and the `stereo` and `sensing` values beyond `1` and `0` (§3.11.12, §3.11.13). The reference project has one amplifier and no microphone.
11. **Variables with Values:** A `0x089B` `0x0898` reply, the `0x089D` datagram, and the `variable_status` bodies for Integer and String variables have not been captured; the reference project declares no variables. Also not established: behaviour when the Boolean-variable list count differs from the number of Boolean variables, variable types above `2`, and the largest accepted `0x089B` `count` (§3.6).
