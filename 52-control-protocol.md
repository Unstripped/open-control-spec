---
title: "Runtime Control Protocol"
section: "52-control-protocol"
version: "0.1.0"
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
| 0x00 | 1 | `frame_type` | uint8 | `0x02` (standard), `0x00` | Frame initiation marker `[CONFIRMED]` |
| 0x01 | 2 | `length` | uint16 LE | `≥ 2` | Length of subsequent payload bytes including `opcode` `[CONFIRMED]` |
| 0x03 | 2 | `opcode` | uint16 LE | Listed in §3 | Command opcode identifier `[CONFIRMED]` |
| 0x05 | `length - 2` | `payload` | bytes | | Command-specific argument data `[CONFIRMED]` |

Total wire size = `length + 1` bytes.

### 2.2 Response Frame Format

Responses from the controller return a 3-byte header prefix, response opcode, status flag, and returned data:

| Offset | Size | Field | Type | Allowed Values | Description |
|---|---|---|---|---|---|
| 0x00 | 1 | `resp_type` | uint8 | `0x00` (query), `0x01` (action ACK) | Response category marker `[CONFIRMED]` |
| 0x01 | 2 | `resp_length` | uint16 LE | `≥ 3` | Response payload size including `resp_opcode` and `status` `[CONFIRMED]` |
| 0x03 | 2 | `resp_opcode` | uint16 LE | | Associated response opcode `[CONFIRMED]` |
| 0x05 | 1 | `status` | uint8 | `0x01` (success), `0x00` (error) | Execution result status `[CONFIRMED]` |
| 0x06 | `resp_length - 3` | `body` | bytes | | Returned query results or transaction metadata `[CONFIRMED]` |

---

## 3. Command Opcode Catalog

### 3.1 Macro Execution (`0x07DA`)

- **Name:** `MACRO_ACTION`
- **Request Opcode:** `0x07DA`
- **Request Payload (16 bytes):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `room_id` | uint32 LE | Target Room identifier |
| 0x04 | 4 | `device_id` | uint32 LE | Target Device identifier |
| 0x08 | 4 | `macro_id` | uint32 LE | Macro or button action index |
| 0x0C | 4 | `flags` | uint32 LE | Action flags / execution parameters |

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

### 3.7 Project File Timestamp Query (`0x0FE5`)

- **Name:** `GET_FILE_TIME`
- **Request Opcode:** `0x0FE5`
- **Request Payload:** None (`length = 2`)
- **Response Opcode:** `0xFFFD`
- **Response Payload (4 bytes):**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `timestamp` | uint32 LE | Unix timestamp of active project files |

---

## 4. Error Handling

1. **Connection Timeout:** If a client fails to transmit data within 10,000 ms, the worker thread terminates the connection.
2. **Invalid Opcode:** The controller ignores unrecognized opcodes or emits an error log and closes the socket.
3. **Invalid Room or Macro ID:** Returns an action ACK with an error code status byte (`status = 0x00`).

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

---

## 6. Open Items

1. **Encrypted Session Mode:** Remotes supporting seed negotiation (`GET_SECURE_DATA_SEED`) can negotiate ephemeral AES session encryption for off-site / cloud transport.
