---
title: "Network Discovery Protocol (UDP 61510 & mDNS)"
section: "50-discovery"
version: "0.5.0"
status: review
confidence: confirmed
---

# 50. Network Discovery Protocol

## 1. Overview

Controllers implement active UDP discovery on **UDP port 61510**. A client sends a discovery request (`DISCOVERY_PROBE`, opcode `0xC350`); a matching controller returns a discovery response (`DISCOVERY_REPLY`, opcode `0xC353`).

The packet opcodes and model identifier are little-endian. Controllers silently ignore a byte-swapped request.

---

## 2. UDP Discovery Frames

### 2.1 Discovery Probe (`DISCOVERY_PROBE` / `0xC350`)

The standard request is 5 bytes:

| Offset | Size | Type | Value | Meaning |
|---|---|---|---|---|
| `0x00` | 1 | `uint8` | `seq` | Sequence number echoed by the response |
| `0x01` | 2 | `uint16_le` | `0xC350` | Discovery opcode; wire bytes `50 C3` |
| `0x03` | 2 | `uint16_le` | model filter | `0x0FFF` selects all models; any value whose low 12 bits are `0xFFF` is treated as a wildcard `[CONFIRMED]`. Any other value MUST equal the controller's model identifier exactly; otherwise the request is ignored `[STATIC]` |

An exact 9-byte request appends:

| Offset | Size | Type | Meaning |
|---|---|---|---|
| `0x05` | 4 | `in_addr` | IPv4 destination for the response |

Clients MUST send either the 5-byte standard request or the 9-byte routed request. The UDP source port MAY be ephemeral.

### 2.2 Discovery Response (`DISCOVERY_REPLY` / `0xC353`)

The response is exactly 55 bytes:

| Offset | Size | Type | Value / Example | Meaning |
|---|---|---|---|---|
| `0x00` | 1 | `uint8` | `seq` | Echoed request sequence |
| `0x01` | 2 | `uint16_le` | `0xC353` | Response opcode; wire bytes `53 C3` |
| `0x03` | 2 | `uint16_le` | `0x8008` | Controller model identifier |
| `0x05` | 6 | bytes | MAC address | Controller hardware MAC address |
| `0x0B` | 4 | `in_addr` | IPv4 address | Current controller IPv4 address |
| `0x0F` | 30 | byte string | description | User description; see §2.2.1 |
| `0x2D` | 10 | byte string | `MRX-15` + `0x0A` | Model name; see §2.2.1 |

The response does not contain netmask, gateway, DNS, firmware version, or 32-byte labels.

#### 2.2.1 Text Fields

Both text fields are raw byte strings copied verbatim from controller configuration text. The controller performs no character-set conversion and declares no encoding `[STATIC]`. Unused bytes are `0x00`.

- **Model name** (`0x2D`): the remainder of the controller's model-name configuration line, including its line terminator. The verified controller sends `MRX-15` followed by `0x0A` and three `0x00` bytes `[CONFIRMED]`. The controller copies a name only when it is at most 9 bytes including the terminator, so a copied name is always followed by at least one `0x00` `[STATIC]`; field contents for a longer configured name are not established.
- **Description** (`0x0F`): the first line of the controller's stored description, at most 30 bytes `[STATIC]`. If that line is shorter than 30 bytes and ends with a newline, the `0x0A` is included; a 30-byte description fills the field with no `0x00` terminator `[STATIC]`. An unset description is all `0x00` `[CONFIRMED]`.

Clients MUST read each field up to its first `0x00` byte or the end of the field, and MUST then remove trailing `0x0A`, `0x0D`, and ASCII whitespace. Observed values are ASCII `[CONFIRMED]`. The encoding of bytes `0x80`–`0xFF` is not established; clients SHOULD decode them as UTF-8 with replacement of invalid sequences and MAY preserve the raw bytes for display or comparison `[INFERRED]`.

### 2.3 Response Routing

- The controller listens on UDP 61510 on all of its IPv4 addresses. It answers requests sent to its unicast address, to the subnet-directed broadcast address, and to the limited broadcast address `255.255.255.255` `[CONFIRMED]`.
- For an exact 9-byte request, the controller sends the response to the IPv4 address at request offset `0x05`.
- For the 5-byte request, the controller sends the response to `255.255.255.255`.
- In both cases, the destination UDP port equals the request's UDP source port, and the response's UDP source port is 61510 `[CONFIRMED]`. Clients MUST listen on the request's source port until the timeout expires.
- Controllers MAY emit duplicate responses; one broadcast probe has produced two to five identical responses `[CONFIRMED]`. Clients SHOULD de-duplicate by MAC address.

These routing rules were verified with both fixed and ephemeral client source ports `[CONFIRMED]`.

On a multi-homed client, the operating system chooses one interface for a datagram addressed to `255.255.255.255`. Clients that must search every attached network SHOULD send a subnet-directed broadcast on each interface, or bind one socket per interface, rather than relying on a single limited-broadcast probe.

---

## 3. mDNS

An mDNS responder may be present on UDP 5353, but `_urc._tcp.local` was not advertised by the verified controller firmware. Clients MUST NOT depend on that service advertisement for controller discovery. Implementations MAY use independently observed mDNS records as an optional hint.

---

## 4. Verification Vectors

### 4.1 Wildcard Request

Transmit to the controller's unicast address, the subnet broadcast address, or `255.255.255.255` on UDP 61510:

```text
01 50 C3 FF 0F
```

The byte-swapped request `01 C3 50 FF 0F` is invalid and receives no response.

### 4.2 Sanitized Example Response

```text
01 53 C3 08 80 02 00 00 00 00 01 C0 00 02 2B 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 4D 52 58
2D 31 35 0A 00 00 00
```

This vector is 55 bytes. It identifies model `0x8008`, a locally administered example MAC, documentation IPv4 address `192.0.2.43`, an empty description, and model name bytes `MRX-15` `0x0A`, which a client reports as `MRX-15` after the trimming in §2.2.1.

---

## 5. Open Items

None.
