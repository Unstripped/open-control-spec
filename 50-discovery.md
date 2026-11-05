---
title: "Network Discovery Protocol (UDP 61510 & mDNS)"
section: "50-discovery"
version: "0.2.0"
status: ratified
confidence: confirmed
circumvention: false
---

# 50. Network Discovery Protocol

## 1. Overview & Architecture

URC Total Control network devices utilize active UDP broadcast discovery managed by the `provisioningd` daemon on **UDP port 61510**, supplemented by multicast DNS (**mDNS**) on UDP port 5353.

*(Note: While legacy documentation sometimes referenced UDP 51535 and multicast `235.123.0.1`, empirical network observation confirms UDP 51535 is the application logging socket and `235.123.0.1` is reserved for legacy streaming players. Controller discovery runs on UDP 61510.)*

---

## 2. UDP Discovery Frames (Port 61510)

All numeric multi-byte fields are little-endian unless specified otherwise.

### 2.1 Discovery Probe (`PACKET_FIND` / `0xC350`)
The programmer or client transmits a UDP broadcast packet to `255.255.255.255:61510`:

| Offset | Size | Type | Value | Meaning |
|---|---|---|---|---|
| `0x00` | 1 | `uint8` | `seq` | Sequence number (incremented per request) |
| `0x01` | 2 | `uint16_be` | `0xC350` | Discovery Opcode (50000) |
| `0x03` | 2 | `uint16_le` | `0x0FFF` | Target Model ID (`0x0FFF` = wildcard / all models) |

*Wire size:* The standard broadcast probe is exactly 5 bytes (`0x00..0x04`). An optional unicast-routed variant is 9 bytes, where bytes `0x05..0x08` contain the requester's IPv4 address `[CONFIRMED]`.

### 2.2 Discovery Response (`PACKET_FIND_ACK` / `0xC353`)
The controller replies directly to the requester on UDP port 61510:

| Offset | Size | Type | Value / Example | Meaning |
|---|---|---|---|---|
| `0x00` | 1 | `uint8` | `seq` | Echoed transaction sequence number |
| `0x01` | 2 | `uint16_be` | `0xC353` | Response Opcode (50003) |
| `0x03` | 2 | `uint16_le` | `0x8008` | Hardware Model ID. MRX-15 product code is `0x8008` (matching `prov_model.ini` and package manifest prefixes `M8008_*`); `0x8000` is the generic MRX base controller family code `[CONFIRMED]`. |
| `0x05` | 4 | `uint32_le` | `0x00000000` | Reserved / flags |
| `0x09` | 6 | `bytes` | `00:1F:B8:00:11:22` | Controller Hardware MAC Address |
| `0x0F` | 4 | `in_addr` | `192.168.1.100` | Current Controller IPv4 Address |
| `0x13` | 4 | `in_addr` | `255.255.255.0` | Subnet Mask |
| `0x17` | 4 | `in_addr` | `192.168.1.1` | Default Gateway IP Address |
| `0x1B` | 4 | `in_addr` | DNS Server | Configured DNS Server IP Address |
| `0x1F` | 32 | `string` | `MRX-15\0...` | Controller Model Name string |
| `0x3F` | 32 | `string` | `MRX-15 Master\0...` | User Description string |

*Wire size:* The response frame wire size is exactly 95 bytes (`0x5F` bytes, spanning offsets `0x00..0x5E`) `[CONFIRMED]`.
---

## 3. Passive mDNS Discovery (Port 5353)

`mdnsd` advertises the controller on multicast group `224.0.0.251:5353`:
- **Service Name:** `_urc._tcp.local`
- **Host Target:** `<hostname>.local`
- **TXT Record Keys:** `model=MRX-15`, `mac=00:1F:B8:00:11:22`, `version=3.0.2022.06101`.

---

## 4. Discovery Verification

A discovery probe broadcasts `PACKET_FIND` (`0xC350`) and unpacks `PACKET_FIND_ACK` (`0xC353`) responses from local controllers to identify controller IP, MAC, and model:

- **Probe packet:** Transmit 5 bytes `01 C3 50 FF 0F` to `255.255.255.255:61510`.
- **Response unpacking:** Validate response opcode `0xC353` and parse network fields according to Table 2.2.
---

## 5. Open Items

None. Discovery frames and fields are completely documented.
