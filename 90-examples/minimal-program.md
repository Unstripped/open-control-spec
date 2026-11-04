---
title: "Minimal Program Worked Example"
section: 90
status: stable
confidence: confirmed
circumvention: false
---

# 90. Minimal Program Worked Example

## 1. Overview & Specification Scope

This document defines the smallest valid, self-consistent program package required to program an MRX controller:
- **Topology:** Exactly one Base Station (`MRX-15`, MAC `00:1F:B8:00:11:22`), one Room ("Living Room", ID `0`), one Device ("Apple TV", dev_id `3`), one variable (`TestVar`, var_id `1`), and bound macro actions.
- **Package Files:**
  1. `mcs.dat`: JSON audio/device routing model (Section 30).
  2. `mcs_info.dat`: JSON system topology and object catalog (Section 30).
  3. `DnData.dn`: Binary controller database with 26 indexed tables (Section 32).
  4. `Link.dat`, `Event.dat`, `Alarm.dat`: Automation and schedule headers (Section 31).
  5. `rms.ini`: RMS configuration file (22 bytes).
  6. `CCS/edit.dat.gz`: Master dealer backup database (Section 33).
  7. Staged release files and manifests (`M0000_*.info`, `M8008_*.info`) (Section 20 & 51).

---

## 2. File Specifications & Derivations

### 2.1 `mcs_info.dat` (System Topology Model)
Compact single-line JSON (`separators=(',', ':')`) defining the base station, client seat 5 (iPhone), and the controlled device:

```json
{"bstations":[{"obj_id":1,"name":"MRX-15 Master","model":"MRX-15"}],"clients":[{"obj_id":5,"name":"iPhone Mobile","model":"IPhoneV2"}],"devices":[{"obj_id":3,"name":"Apple TV","model":"DBMODEL","db":{"type":"IR","category":"AUX","brand":"Apple","model":"APPLE TV(4GEN)"}}]}
```

### 2.2 `mcs.dat` (Zone & Matrix Topology Model)
Compact single-line JSON mapping the connected devices, audio feedback, and zone assignments:

```json
{"con_devs":[{"id":3,"name":"Apple TV","db":{"type":1,"builtins":[],"zones":[{"idx":0,"funcs":[{"id":530,"name":"Power"}]}]}}],"special_jump":[]}
```

### 2.3 `rms.ini` (RMS Remote Management Configuration)
Standard 22-byte INI configuration:
```ini
[RMS]
ENABLE=0
ID=
```

### 2.4 `DnData.dn` (Binary Runtime Database)
- **Primary Header (56 B):**
  - Offset `0x00`: First-table offset minus 4 (`uint32_le`).
  - Offset `0x04`: 8-byte generation timestamp identifier (`uint64_le`).
  - Offset `0x0C`: Target model string (32 bytes ASCII, `MRX-15\0`).
  - Offset `0x2C`: Total file size in bytes (`uint32_le`).
  - Offset `0x30`: Database format version `0x00117000` (`uint32_le`).
  - Offset `0x34`: Firmware operational type `0` (`uint32_le` for Master DB).
- **Project Metadata String:** Project ID `1` (`uint32_le`), Name length `16` (`uint16_le`), Project Name `Minimal Program\0`.
- **Table Directory:** 26 little-endian 32-bit table offsets.
- **Table 0 (`ConfigObjTable`):** 5-byte empty table form `00 00 00 00 00`.
- **Table 5 (`ConnectedDeviceTable`):** Exactly 1 device record for Apple TV (dev_id `3`, model `DBMODEL`, IR transport, base station 1 port 1, 0 commands).
- **Table 8 (`VariableTable`):** Exactly 1 integer variable: var_id `1`, name `TestVar`, initial value `0`, min `0`, max `100`.
- **Table 10 (`RoomObjTable`):** Exactly 1 room record ("Living Room", ID `0`):
  - Inner device `3` ("Apple TV") carrying Button `0` with macro action `[OP_VAR_STATE(1, 7)]`.
  - Room Power On macro (`pwr_on_btn`) with action `[OP_VAR_STATE(1, 42)]`.
  - Room Power Off macro (`pwr_off_btn`) with action `[OP_VAR_STATE(1, 0)]`.
- **All other tables (1..4, 6..7, 9, 11..25):** 8-byte empty headers (`04 00 00 00 00 00 00 00`).

### 2.5 Automation Tables (`Link.dat`, `Event.dat`, `Alarm.dat`)
- `Link.dat`: 8-byte header (`0x00000000 0x00000000`, 0 linked variables).
- `Event.dat`: 9-byte header (`0x00000000 0x00000000 0x00`, version 1.0, 0 events) + 2-byte footer (`0x0000`).
- `Alarm.dat`: 9-byte header (`0x00000000 0x00000000 0x00`, version 3.0, 0 alarms).

### 2.6 Master Project Database (`CCS/edit.dat.gz`)
RFC 1952 gzip archive containing `edit.dat` (header magic `1917`, schema `2109440`, 1 room, 1 device, sentinel `END_FILE_CHECK`), compressed with deterministic `mtime=0`.

---

## 3. Staged Release Form

When packaged for controller staging via FTP (`spec/51-upload-protocol.md`), the minimal program produces three `.info` manifests and corresponding content-addressed artifacts:

| Staged Name Pattern | Destination on Controller | Owner Prefix | Target Address |
|---|---|---|---|
| `M8008_M<MAC>0000_C<token>_DnData.dn` | `/remote/Store/data/DnData.dn` | `M8008` | `M<MAC>0000` |
| `M8008_A0000000000000000_C<token>_rms.ini` | `/remote/Store/data/rms.ini` | `M8008` | `A0000000000000000` |
| `M0000_A0000000000000000_C<token>_edit.dat.gz` | `/remote/Store/CCS/edit.dat.gz` | `M0000` | `A0000000000000000` |

Staging manifests:
1. `M8008_M<MAC>0000_T<time>.info`: lists `DnData.dn`.
2. `M8008_A0000000000000000_T<time>.info`: lists `rms.ini`.
3. `M0000_A0000000000000000_T<time>.info`: lists `edit.dat.gz`.

---

## 4. Expected Hardware Behavior

After deployment and activation on the controller, runtime queries to TCP port 51510 must yield the following exact sequence:

| Step | Action / Command | Sent Frame / Parameters | Expected Response / Status | Variable 1 State |
|---|---|---|---|---|
| 1 | Query initial state | `VARIABLE_CMD_GET_TABLE` (`0x089A`) | Response `0x0898`, count = 1 | `var1 = 0` |
| 2 | Execute Device Macro | `MACRO_ACTION` (`0x07DA`), Room 0, Dev 3, Macro 0 | ACK `0x07DB`, status `0x01` | `var1 = 7` |
| 3 | Room Power On | `ROOMS_CMD_ROOM_POWER_ON` (`0x0012`), Room 0 | ACK `0x07DB`, status `0x01` | `var1 = 42` |
| 4 | Room Power Off | `ROOMS_CMD_ROOM_POWER_OFF` (`0x000D`), Room 0 | ACK `0x07DB`, status `0x01` | `var1 = 0` |
| 5 | Invalid Room Macro | `MACRO_ACTION` (`0x07DA`), Room 5, Dev 0, Macro 0 | ACK `0x07DB`, status `0x00` (error) | `var1 = 0` |
