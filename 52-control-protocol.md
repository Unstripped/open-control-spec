---
title: "Runtime Control Protocol"
section: "52-control-protocol"
version: "0.8.0"
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
| 0x00 | 1 | `frame_type` | uint8 | `0x02` (standard), `0x00` | Frame initiation marker. `0x02` indicates standard framed request; `0x00` indicates legacy direct unproxied request `[CONFIRMED]` |
| 0x01 | 2 | `length` | uint16 LE | `≥ 2` | Length of subsequent payload bytes including `opcode` `[CONFIRMED]` |
| 0x03 | 2 | `opcode` | uint16 LE | Listed in §3 | Command opcode identifier `[CONFIRMED]` |
| 0x05 | `length - 2` | `payload` | bytes | | Command-specific argument data `[CONFIRMED]` |

Total wire size = `length + 3` bytes (1-byte `frame_type` + 2-byte `length` + `length` bytes of payload).

Payload length checking is opcode-specific. Some opcodes ignore surplus or missing payload bytes; others reject a wrong length with the generic negative response (§2.3). Clients MUST send exactly the payload size documented for each opcode.

### 2.1.1 Request Prefix Byte

The `MACRO_TRIGGER` (`0x07DA`) and `SET_LEVEL_ABSOLUTE` (`0x07ED`) payloads begin with a 1-byte `prefix`. Their command fields start at payload offset `0x01` `[CONFIRMED]`. Whether other opcodes carry this byte is not established.

- Clients MUST send `prefix = 0x00`.
- Values `0x20`–`0x3F` select a separate session-establishment path. For any such value, with either opcode, the controller does not perform the command. It replies with the fixed 54-byte frame `00 33 00 72 17 22` followed by a 48-byte body, then closes the connection `[CONFIRMED]`. `0x1772` is not a reply to the command, and that path is outside this specification.
- All other values tested had no observed effect: the full range for `0x07ED`, and `0x00`–`0x04`, `0x08`, `0x10`, `0x40`, and `0x80` for `0x07DA` `[CONFIRMED]`.

A client that omits the prefix shifts every field by one byte. The trigger code's low byte is then read as `prefix`, so a `button_id` whose low byte is `0x20`–`0x3F` produces the `0x1772` frame described above.

### 2.2 Response Frame Format

Responses from the controller return a 3-byte header prefix, response opcode, status byte, and returned data:

| Offset | Size | Field | Type | Observed Values | Description |
|---|---|---|---|---|---|
| 0x00 | 1 | `resp_type` | uint8 | `0x00`, `0x01` | Response category marker. The value is fixed per response path, not per outcome (for example, `0x07DB` uses `0x01`; `0x07EE` and `0xFFFD` use `0x00`). Clients MUST NOT use it to decide success `[CONFIRMED]` |
| 0x01 | 2 | `resp_length` | uint16 LE | `≥ 3` | Response payload size including `resp_opcode` and `status` `[CONFIRMED]` |
| 0x03 | 2 | `resp_opcode` | uint16 LE | | Response opcode `[CONFIRMED]` |
| 0x05 | 1 | `status` | uint8 | `0x01`, `0x02` | See below `[CONFIRMED]` |
| 0x06 | `resp_length - 3` | `body` | bytes | | Returned query results or transaction metadata `[CONFIRMED]` |

Total wire size = `resp_length + 3` bytes.

**Status byte.** `0x01` is returned on successful replies and also on several rejections (for example the generic negative response `0xFFFE` in §2.3 and room-power replies for unknown rooms). `0x02` has been observed only with `resp_opcode = 0xFFFE` when a room-audio command (§3.11) names an unknown room `[CONFIRMED]`. No examined reply path emits `0x00` `[STATIC]`. The outcome of a request is conveyed primarily by `resp_opcode`: clients MUST treat `0xFFFE` as a negative result regardless of `status`, and SHOULD treat any `status` other than `0x01` as a failure.

Communication is synchronous and half-duplex per connection: a client transmits one framed request and awaits its matching response before issuing any permitted subsequent request. Some requests produce more than one response frame (§3.1); clients MUST parse all frames by `resp_length` and MUST NOT assume one frame per request. The controller never transmits unsolicited asynchronous event frames over this socket; asynchronous status and macro progress events are emitted over UDP 61610/61611, UDP 57002, or MQTT TCP 1883.

### 2.3 Generic Responses (`0xFFFD`, `0xFFFE`)

`0xFFFD` is a generic positive response opcode used by several queries (§3.7–§3.9). `0xFFFE` is a generic negative response opcode. It is not specific to one request. Observed and static sources of `0xFFFE`:

| Condition | Frame | Connection effect | Confidence |
|---|---|---|---|
| Controller has not finished system initialization when the connection is accepted | `00 03 00 FE FF 01` | Closed by controller | `[STATIC]` |
| No complete request arrives within 10,000 ms of connection establishment | `01 03 00 FE FF 01` | Closed about 3 s later | `[CONFIRMED]` |
| `0x07ED` request with a payload other than 19 bytes | `01 03 00 FE FF 01` | — | `[CONFIRMED]` |
| Room-audio command (§3.11) naming an unknown room | `00 03 00 FE FF 02` | — | `[CONFIRMED]` |
| `NETWORK_STATUS_QUERY` while the controller is not online (§3.7) | `00 03 00 FE FF 01` | Terminal | `[STATIC]` |

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

- `MACRO_TRIGGER` carries no zone, flags, or press/hold/release field. This section does not establish a way to address one zone of a multi-zone amplifier through `0x07DA` alone. A trigger code reaches a zone only through the project's button and macro definitions. The controller times a press itself (§3.1.4).
- Function IDs `275` (volume up), `276` (volume down), `277` (mute toggle), `552` (mute on), and `553` (mute off) appear in device function lists for audio zones `[STATIC]`. The mapping from a device function ID to a trigger code is defined by the project model, not by this protocol.
- In reference projects, source selection is performed by controller-side macros attached to room menu tiles, not by a published input function ID `[STATIC]`. Earlier revisions cited function IDs `1001` and `1002` as inputs; those IDs were not observed in any reference project and are withdrawn.
- Room-addressed volume and mute commands are listed in §3.11; none has been observed to succeed (§6). No trigger code accepting `0x07ED` is known in the reference project (§3.2).

#### 3.1.2 Responses

The controller sends a sequence of frames on the request's connection `[CONFIRMED]`:

1. `TRIGGER_ACK` (`0x07DB`), 8 bytes: `01 05 00 DB 07 01` followed by a 2-byte body. The body is controller state, not a client-meaningful handle; observed value `01 02` `[CONFIRMED]`. Clients SHOULD ignore its contents.
2. When the trigger code resolves to an action:
   1. `00 04 00 DD 07 01 01` (`resp_opcode = 0x07DD`, body `01`): sent when the action starts.
   2. Zero or more level reports (`0x07E1`, §3.1.3).
   3. `00 04 00 DD 07 01 00` (`0x07DD`, body `00`): sent when the action ends.
   4. `00 03 00 DC 07 01` (`resp_opcode = 0x07DC`, no body).
3. Otherwise `0x07DC` follows `0x07DB` directly. Observed for unknown trigger codes, for trigger code `0`, and for one valid code that produced no observed device traffic `[CONFIRMED]`.

For valid buttons, `0x07DC` arrived about 0.5 s after the request for a mute toggle, 1.0 s for a networked speaker's volume step, and 1.4 s for an amplifier volume step `[CONFIRMED]`. `0x07DC` was the last frame in every observed sequence. Clients MUST keep reading until `0x07DC`; `MACRO_TRIGGER` is terminal after it (§4). Closing the connection earlier does not cancel the action: a client that closed right after `0x07DD` `01` saw the same device commands as one that held the connection open `[CONFIRMED]`. No frame reports whether the device carried the action out. Clients MUST NOT treat `0x07DC` as an error, and MUST NOT treat `0x07DB` alone as proof that a button ran.

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

#### 3.1.4 Volume Steps

- A volume button sent through `0x07DA` is a press the controller holds for a period it chooses, not a single step. During the hold it sends the device's volume command repeatedly; the client sends nothing further `[CONFIRMED]`.
- In the reference project, one volume-up or volume-down press on a device page whose volume is bound to the room's amplifier zone sent the amplifier's relative volume command six times over about 1.25 s. The count was the same whether the client closed the connection after `0x07DD` or held it open `[CONFIRMED]`. A mute toggle on the same page sent one command `[CONFIRMED]`.
- The size of each step belongs to the device. Neither this protocol nor the project sets it. The reference amplifier moves 2 levels per relative command, so one press changed the level by 12 `[CONFIRMED]`.
- Repeat timing comes from per-command settings in the project's device database: repeat count, ramp start, and ramp speed (`32-download-database` §5.2.1) `[STATIC]`. The reference amplifier's volume commands carry repeat count `1`, ramp start `300`, and ramp speed `100`. How the controller derives the number of sends from these settings and its hold period is not established. A button whose macro step names the device command directly carries its own copy of these settings, so its count can differ.
- Clients MUST NOT assume a fixed step count or step size. Clients SHOULD read the resulting level from `0x07E1` reports (§3.1.3) where the device provides them.

### 3.2 Absolute Volume Control (`0x07ED`)

- **Name:** `SET_LEVEL_ABSOLUTE`
- **Request Opcode:** `0x07ED`
- **Description:** Sets an absolute volume level and mute state through a device's volume control `[STATIC]`.
- **Targets:** The controller contains a volume routine that sets a level through a device's scripted two-way driver, addressed by zone index `[STATIC]`. That `0x07ED` reaches devices through this routine is a hypothesis; the path from the `0x07ED` handler to it has not been traced `[INFERRED]`. Which project elements or devices qualify is not established. Neither `core_model.json` nor the button catalog marks a trigger as accepting `0x07ED`, and no accepting trigger is known in the reference project. There, `0x07ED` with the volume-up trigger of a device page bound to the room's amplifier, and with the volume-up trigger of a device that has a two-way driver, sent no command to either device and left the amplifier's level unchanged `[CONFIRMED]`. Because the reply echoes any trigger code (below), a client cannot find an accepting trigger from the reply. Clients SHOULD use `0x07ED` only with a trigger a user has confirmed against the device, and otherwise step volume with `0x07DA` (§3.1.4).
- **Request Payload (19 bytes, `length = 21`):** Any other payload length is rejected with `01 03 00 FE FF 01` `[CONFIRMED]`.

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

The body is request payload offsets `0x01`–`0x11` verbatim: `prefix` and `reserved` are not echoed `[CONFIRMED]`. It does not report the resulting device level, and it is returned even for a trigger code that does not exist in the project `[CONFIRMED]`. A body equal to the request's fields therefore confirms framing only. `0x07ED` is terminal (§4).

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
- **Response:** `TRIGGER_ACK` (`0x07DB`), 8 bytes, as in §3.1.2 `[STATIC]`. Not exercised live.
- **Connection Effect:** Not fully established. Clients MUST treat it as terminal (§4 rule 4).

### 3.5 Room Power On (`0x0012`)
- **Name:** `ROOM_POWER_ON`
- **Request Opcode:** `0x0012`
- **Request Payload (4 bytes):** `room_id` (uint32 LE)
- **Response:** `00 07 00 14 00 01` followed by a 4-byte body (`resp_opcode = 0x0014`) `[CONFIRMED]`. The body was `00 00 00 00` for an unknown room; its meaning for valid rooms is not established.
- **Connection Effect:** Terminal (§4) `[CONFIRMED]`.

### 3.6 System Variable Table Query (`0x089A`)
- **Name:** `VARIABLE_TABLE_QUERY`
- **Request Opcode:** `0x089A`
- **Request Payload:** None (`length = 2`)
- **Response Opcode:** `0x0898`
- **Response Payload ($4 + 5 \times N$ bytes):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `count` | uint32 LE | Number of variable status records returned ($N$) `[CONFIRMED]` |
| 0x04 | $5 \times N$ | `entries` | record[$N$] | Array of 5-byte variable status records `[CONFIRMED]` |

Each 5-byte variable status entry consists of:

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| `+0x00` | 4 | `var_id` | uint32 LE | System variable identifier `[CONFIRMED]` |
| `+0x04` | 1 | `value` | uint8 | Current runtime value (0..255) `[CONFIRMED]` |

### 3.7 Network Status Query (`0x0FE1`)
- **Name:** `NETWORK_STATUS_QUERY`
- **Request Opcode:** `0x0FE1`
- **Request Payload:** None (`length = 2`)
- **Response:** `00 03 00 FD FF 01` (online) or `00 03 00 FE FF 01` (not online). No body. The status byte is `0x01` in both cases; the response opcode carries the result `[STATIC]`. The online frame is `[CONFIRMED]`.
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

### 3.11 Room Audio Commands (Room-Addressed Volume and Mute)

These commands address a room's audio zone directly. Opcode assignments and the volume and mute layouts are `[STATIC]`. For each, an unknown room produces `00 03 00 FE FF 02` `[CONFIRMED]`. Successful responses and connection lifetime have not been observed; clients MUST treat these commands as terminal (§4 rule 4).

Each function has three addressing forms:

| Function | Short form | Extended form | Connected-device form |
|---|---|---|---|
| Get zone audio state | `0x0004` | `0x007C` | `0x0096` |
| Set volume | `0x0005` | `0x007D` | `0x0097` |
| Set mute | `0x0006` | `0x007E` | `0x0098` |
| Set balance | `0x0007` | `0x007F` | `0x0099` |
| Set treble | `0x0008` | `0x0080` | `0x009A` |
| Set bass | `0x0009` | `0x0081` | `0x009B` |

Payload layouts established so far:

| Opcode | Payload |
|---|---|
| `0x0005` | `room` (uint8), `value` (uint8) |
| `0x007D` | `room_id` (uint32 LE), `value` (uint8) |
| `0x0097` | `con_dev_id` (uint32 LE, audio device identifier from `mcs.dat`), `zone_index` (uint8), `value` (uint8) |
| `0x0006` | `room` (uint8), `mute` (uint8) |
| `0x007E` | `room_id` (uint32 LE), `mute` (uint8) |
| `0x0098` | `con_dev_id` (uint32 LE), `zone_index` (uint8), `mute` (uint8) |

`value` is the zone's raw level, not a percentage. A client converting from a percentage SHOULD send `min + (max − min) × percent / 100`, using the zone's configured `min_vol` and `max_vol` (Section 30) `[STATIC]`. The `mute` values are `0` (unmuted) and `1` (muted) `[INFERRED]`. Layouts for the get, balance, treble, and bass commands are not yet established.

---

## 4. Connection Lifecycle & Error Handling

1. **First Request Deadline:** The controller waits up to 10,000 ms after accepting a connection for the first request. If none arrives, it sends `01 03 00 FE FF 01` and closes the connection about 3 s later `[CONFIRMED]`.
2. **One Request at a Time:** Clients MUST consume every response frame for a request (§2.2) before issuing a subsequent request on a reusable connection.
3. **Terminal Commands:** After its final response frame to `MACRO_TRIGGER` (`0x07DA`), `SET_LEVEL_ABSOLUTE` (`0x07ED`), `ROOM_POWER_ON` (`0x0012`), `NETWORK_STATUS_QUERY` (`0x0FE1`), or `VARIABLE_TABLE_QUERY` (`0x089A`), the controller waits up to 10,000 ms and then closes the socket `[CONFIRMED]`. `ROOM_POWER_OFF` (`0x000D`) follows the same path `[STATIC]`. If the client sends another frame during that wait, those bytes are consumed and discarded, and the controller closes the connection without a response `[CONFIRMED]`. Clients MUST open a new connection for the next request, and SHOULD close their side as soon as the final response frame is read rather than waiting for the controller.
4. **Portable Client Rule:** Unless an opcode is explicitly documented here as reusable, clients MUST treat each request/response transaction as terminal and reconnect for the next request.
5. **Reusable Query:** `CONTROLLER_TIME_QUERY` (`0x0FE6`) returns without closing the socket. Further requests MAY follow after its complete response `[CONFIRMED]`. After a reusable response, the controller polls the socket in 3,000 ms receive intervals and does not close an idle connection on that path `[STATIC]`; a connection left idle for 125 s after `0x0FE6` remained open and answered another `0x0FE6` `[CONFIRMED]`. No upper idle limit is specified. Clients MUST NOT rely on an indefinite idle lifetime and SHOULD close connections they no longer need.
6. **Concurrency:** Port 51510 listens with a backlog of 64. Connecting a new client does not disconnect another active client `[CONFIRMED]`. Connections beyond 64 active clients are closed without a response `[STATIC]`.
7. **Startup:** If the controller has not finished system initialization, it answers an accepted connection with `00 03 00 FE FF 01` and closes it `[STATIC]`. Clients SHOULD retry after a delay.
8. **Invalid Opcode:** An unrecognized opcode receives no response, and the connection stays open; a subsequent `0x0FE6` on the same connection is answered `[CONFIRMED]`. Clients SHOULD apply a response timeout of a few seconds and then close the connection.
9. **Invalid Room, Object, or Trigger:** No uniform error reply exists. `0x07DA` acknowledges unknown trigger codes with `0x07DB` followed by `0x07DC` (§3.1.2); `0x07ED` echoes them normally; `0x000D` and `0x0012` return their normal frames for unknown rooms; room-audio commands (§3.11) return `00 03 00 FE FF 02` `[CONFIRMED]`. Clients MUST validate identifiers against the project model before sending.

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

### 5.4 Wrong-Length Volume Request

Request with a 17-byte payload (`length = 19`):
```
02 13 00 ed 07 13 01 02 00 00 00 00 00 64 00 00 00 2d 00 00 00 00
```
Expected response:
```
01 03 00 fe ff 01
```

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

---

## 6. Open Items

1. **Zone Addressing Through Trigger Codes:** How a client addresses one zone of a multi-zone amplifier through `0x07DA` or `0x07ED` trigger codes, beyond the project-defined buttons, is not established.
2. **Hold Period and Release:** How the controller chooses the hold period and send count for a press (§3.1.4), and whether a client can extend or end a hold early. Frames a client sends during a hold are untested.
3. **`0x07DC` / `0x07DD` Frames:** Whether `0x07DD` bodies other than `01` and `00` exist, and the meaning of the unexplained `0x07E1` bytes (§3.1.3).
4. **House Off Lifecycle:** Post-response connection behavior of `0x000E`.
5. **Room Audio Commands:** Successful responses, lifecycle, and the layouts of the get, balance, treble, and bass forms in §3.11. Short-form and extended-form set-volume requests naming the reference room linked to the amplifier (`room` = `2`), with and without a leading `0x00` byte, each returned `00 03 00 FE FF 02` `[CONFIRMED]`. Whether `room` uses a different numbering is not established.
6. **Controller-to-Client Jump:** When a controller-side macro contains a page jump, the controller pushes `0x07DF` to the client. Its payload layout and delivery path are not established.
7. **Prefix Byte Scope:** Whether opcodes other than `0x07DA` and `0x07ED` carry the §2.1.1 prefix. The §3.11 payload layouts are `[STATIC]` and have not been checked for it.
8. **`0x07ED` Targets:** Which devices or project elements accept `0x07ED`, and how a client can identify their trigger codes (§3.2).
