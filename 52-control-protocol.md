---
title: "Runtime Control Protocol"
section: "52-control-protocol"
version: "0.2.0"
status: review
circumvention: false
---

# Runtime Control Protocol

## 1. Scope

The runtime control protocol operates over a persistent TCP connection to port 51510 on the primary automation controller (`BS.exe`). It is used by user interfaces, remotes, test tools, and programmers to query system status, synchronize variables, and trigger macro and device actions.

Confidence tags follow Section 00 §4.5 (`[CONFIRMED]`, `[STATIC]`, `[INFERRED]`).

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
### 2.2 Response Frame Format

Responses from the controller return a 3-byte header prefix, response opcode, status flag, and returned data:

| Offset | Size | Field | Type | Allowed Values | Description |
|---|---|---|---|---|---|
| 0x00 | 1 | `resp_type` | uint8 | `0x00` (query), `0x01` (action ACK) | Response category marker `[CONFIRMED]` |
| 0x01 | 2 | `resp_length` | uint16 LE | `≥ 3` | Response payload size including `resp_opcode` and `status` `[CONFIRMED]` |
| 0x03 | 2 | `resp_opcode` | uint16 LE | | Associated response opcode `[CONFIRMED]` |
| 0x05 | 1 | `status` | uint8 | `0x01` (success), `0x00` (error) | Execution result status `[CONFIRMED]` |
| 0x06 | `resp_length - 3` | `body` | bytes | | Returned query results or transaction metadata `[CONFIRMED]` |

Total wire size = `resp_length + 3` bytes.

The communication model on TCP 51510 is strictly synchronous and half-duplex per connection: a client transmits one framed request and awaits the controller's matching response before issuing the subsequent request. The controller never transmits unsolicited asynchronous event frames over this socket; asynchronous status and macro progress events are emitted over UDP 61610/61611, UDP 57002, or MQTT TCP 1883.
---

## 3. Command Opcode Catalog

### 3.1 Macro Execution (`0x07DA`)

- **Name:** `MACRO_ACTION`
- **Request Opcode:** `0x07DA`
- **Request Payload (16 bytes):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `room_id` | uint32 LE | Target Room identifier (`DnData.dn` Table 10 / `core_model.json`). When triggering a device action (`device_id != 0`), `room_id = 0` is valid and accepted; the controller resolves the assigned room automatically `[CONFIRMED]`. |
| 0x04 | 4 | `device_id` | uint32 LE | Target Device identifier (`mcs.dat` / `DnData.dn` Table 5). Set to `0` for room-level or house-level macros `[CONFIRMED]`. |
| 0x08 | 4 | `macro_id` | uint32 LE | Action or function index: for `device_id != 0`, corresponds to the discrete function ID (`funcs[].id` in `mcs.dat` and `DeviceObjTable`); for `device_id = 0`, corresponds to the room macro ID or trigger code (`TriggerCodeTable`) `[CONFIRMED]`. |
| 0x0C | 4 | `flags` | uint32 LE | Action flags: `0x0000` = single trigger, `0x0001` = Button Press / Macro Start, `0x0002` = Button Release / Macro Stop (ends repeating commands or volume ramps), `0x0003` = Button Hold / Repeat `[CONFIRMED]`. |

- **Response Opcode:** `0x07DB`
- **Response Payload (2 bytes):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 2 | `queue_handle` | uint16 LE | Internal action queue dispatch handle |

### 3.2 Room Power Off (`0x000D`)

- **Name:** `ROOMS_CMD_ROOM_POWER_OFF`
- **Request Opcode:** `0x000D`
- **Request Payload (4 bytes):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `room_id` | uint32 LE | Room identifier to power off |

- **Response Opcode:** `0x07DB` (Action ACK, same format as §3.1)

### 3.3 House Off (`0x000E`)

- **Name:** `ROOMS_CMD_HOUSE_OFF`
- **Request Opcode:** `0x000E`
- **Request Payload:** None (`length = 2`)
- **Response Opcode:** `0x07DB` (Action ACK, same format as §3.1)

### 3.4 Room Power On (`0x0012`)

- **Name:** `ROOMS_CMD_ROOM_POWER_ON`
- **Request Opcode:** `0x0012`
- **Request Payload (4 bytes):** `room_id` (uint32 LE)
- **Response Opcode:** `0x07DB` (Action ACK, same format as §3.1)

### 3.5 System Variable Table Query (`0x089A`)

- **Name:** `VARIABLE_CMD_GET_TABLE`
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

### 3.6 Network Status Query (`0x0FE1`)

- **Name:** `GET_NET_STATUS`
- **Request Opcode:** `0x0FE1`
- **Request Payload:** None (`length = 2`)
- **Response Opcode:** `0xFFFD`
- **Response Payload:** None (Status byte `0x01` indicates controller is online and ready).

### 3.7 Macro Resource Type Query (`0x0FE5`)

- **Name:** `GET_MACRO_COMMAND_RESOURCE_TYPE`
- **Request Opcode:** `0x0FE5`
- **Request Payload (10 bytes, `length = 12`):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 2 | `button_id` | uint16 LE | Button index |
| 0x02 | 4 | `device_id` | uint32 LE | Device object identifier |
| 0x06 | 4 | `room_id` | uint32 LE | Room identifier |

- **Response Opcode:** `0xFFFD`
- **Response Payload:** Returns resource binding classification (e.g. standard IR, Universal Parser, DMS, HDA volume punch-through) `[CONFIRMED]`. If issued with an empty payload, returns an unpopulated status response (`00 03 00 fd ff 01`).

### 3.8 Project File Timestamp & Identity Query (`0x0FE6`)

- **Name:** `GET_FILE_TIME`
- **Request Opcode:** `0x0FE6`
- **Request Payload:** None (`length = 2`)
- **Response Opcode:** `0xFFFD`
- **Response Payload (36 bytes, `resp_length = 39` [0x27] or 51 [0x33]):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `timestamp` | uint32 LE | Current controller system / RTC Unix timestamp `[CONFIRMED]`. |
| 0x04 | 32 | `project_token` | bytes | 32-byte active project identity hash / token loaded from the active database `[CONFIRMED]`. |

Clients poll `0x0FE6` to detect project deployments. While the timestamp field reports the controller's runtime clock, the 32-byte `project_token` reflects the active project database identity. When the returned project token differs from the client's locally cached token, the client invalidates its local state and retrieves updated project models.

### 3.9 Client Authentication & Licensing (`0x1389`)

- **Name:** `CLIENT_LOGIN_REQ`
- **Request Opcode:** `0x1389`
- **Request Payload (13 bytes, total `length = 15`):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `client_id` | uint32 LE | Client seat identifier (matching Table 4 `ClientTable` in `DnData.dn`) |
| 0x04 | 9 | `padding` | bytes | Reserved / zero padding bytes |

- **Response Opcode:** `0x138C` (`CLIENT_LOGIN_ACK`) on authorization, or `0x13EB` (`CLIENT_LOGIN_NACK`) on rejection.
- **Response Payload (5 bytes on ACK, `resp_length = 5`):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 2 | `auth_id` | uint16 LE | Allocated session authorization handle (typically `1`) |

#### Authentication Enforcement Rules
1. **Optional for Control Protocol:** Session login (`0x1389`) is an application-level license validation mechanism required only by commercial URC mobile applications (iOS/Android). The controller automation core does NOT require `0x1389` before processing `0x07DA`, `0x000D`, `0x000E`, `0x0012`, `0x089A`, `0x0FE1`, or `0x0FE6` `[CONFIRMED]`. Clean-room automation clients MAY omit `0x1389` entirely and execute control commands immediately upon TCP connection establishment.
2. **Concurrency & Re-login:** When `0x1389` is called, the controller tracks client sessions by hardware MAC address (supporting up to 200 active client records). If multiple client devices connect using the same seat ID, each is accepted as long as the seat is licensed in Table 4. If an existing client reconnects from the same MAC address, its session is updated and re-acknowledged without disconnecting other clients `[CONFIRMED]`.

---

## 4. Error Handling

1. **Connection Inactivity Timeout:** If a client transmits no data for 10,000 ms, the worker thread terminates the connection `[CONFIRMED]`. Clients maintain persistent connections by periodically issuing a lightweight query (e.g. `GET_NET_STATUS` `0x0FE1`, 5 bytes on wire) every 5,000 ms as a keepalive probe.
2. **Concurrency & Connection Limits:** Port 51510 listens with a connection backlog of 64 (`listen(fd, 0x40)`). The controller spawns an independent worker thread for each accepted connection. Up to 64 concurrent TCP client connections are supported simultaneously. Connecting a new client does not disconnect existing active sessions `[CONFIRMED]`.
3. **Invalid Opcode:** The controller ignores unrecognized opcodes or emits an error log and closes the socket.
4. **Invalid Room or Macro ID:** Returns an action ACK with an error code status byte (`status = 0x00`).

---

## 5. Test Vectors

### 5.1 Status Query (`GET_NET_STATUS`)

Request bytes (5 bytes):
```
02 02 00 e1 0f
```
Expected response (6 bytes):
```
00 03 00 fd ff 01
```

### 5.2 Macro Action (`MACRO_ACTION` Room 0, Device 0, Macro 0)

Request bytes (21 bytes):
```
02 12 00 da 07 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```
Expected response (8 bytes):
```
01 05 00 db 07 01 01 02
```

### 5.3 Fragmented TCP Read (Split Frame Handling)

When receiving framed responses over stream sockets, clients MUST handle frames split across multiple `recv()` calls.

**Scenario:** Response to `GET_NET_STATUS` (total 6 bytes `00 03 00 fd ff 01`) arriving in two segments:

Read 1 (3 bytes - Header):
```
00 03 00
```
*Client buffer accumulates 3 bytes; inspects `resp_length = 0x0003`; calculates total expected size = $3 + 3 = 6$ bytes; awaits remaining 3 bytes.*

Read 2 (3 bytes - Payload & Status):
```
fd ff 01
```
*Client buffer reaches 6 bytes; frame is dispatched to opcode handler `0xFFFD` with status `0x01` (Success).*

### 5.4 Coalesced TCP Read (Multiple Responses in Single Read)

When issuing multiple synchronous requests across a fast local loop, or reading buffered data, multiple framed responses MAY arrive in a single `recv()` call.

**Scenario:** A 14-byte TCP buffer containing a 6-byte `GET_NET_STATUS` response immediately followed by an 8-byte `MACRO_ACTION` ACK response:

Received byte stream (14 bytes):
```
00 03 00 fd ff 01 01 05 00 db 07 01 01 02
```

Parsing sequence:
1. **Frame 1:**
   - Offset `0x00`: `resp_type = 0x00`, `resp_length = 0x0003` (total wire length = 6 bytes).
   - Bytes slice `[0..6]`: `00 03 00 fd ff 01` (Opcode `0xFFFD`, Status `0x01`).
2. **Frame 2:**
   - Offset `0x06`: `resp_type = 0x01`, `resp_length = 0x0005` (total wire length = 8 bytes).
   - Bytes slice `[6..14]`: `01 05 00 db 07 01 01 02` (Opcode `0x07DB`, Status `0x01`, Queue handle `0x0201`).

---

## 6. Open Items

1. **Encrypted Session Mode:** Remotes supporting seed negotiation (`GET_SECURE_DATA_SEED`) can negotiate ephemeral AES session encryption for off-site / cloud transport.
