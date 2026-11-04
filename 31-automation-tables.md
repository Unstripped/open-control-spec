---
title: "Automation & Scheduling Tables: Link.dat, Event.dat, Alarm.dat, and Feedback Descriptors"
status: complete
confidence: confirmed
circumvention: false
---

# Automation & Scheduling Tables

This section specifies the binary table formats for room linking, event scheduling, alarm clocks, and audio zone feedback descriptors in URC Total Control MRX processors.

All three scheduling tables (`Link.dat`, `Event.dat`, `Alarm.dat`) reside in `/remote/Store/data/`. They are loaded at boot by the main control daemon (`BS.exe`) and modified when clients adjust room groups, schedules, or alarms over the network. Audio feedback descriptors reside in `/remote/Store/data/FB/` and are unpacked from `FB.tar`.

---

## 1. Room Link Table (`Link.dat`)

The room link table defines grouped rooms/zones for synchronized volume, power, and audio distribution.

### 1.1 File Header

The file begins with an 8-byte header:

| Offset | Size | Type | Description |
|---|---|---|---|
| `0x00` | 4 | `uint32_le` | **Total Size**: Byte size of entire table starting from offset `0x04` (`4 + payload_bytes`). |
| `0x04` | 4 | `uint32_le` | **Link Count**: Number of room link records in the table ($N \le 256$). If 0, parsing completes. |

### 1.2 Room Link Record

When `link_count > 0`, $N$ variable-length records follow consecutively:

| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | **Record Size**: Total length of this link object record ($22 + \text{name\_len} + 4 \times \text{room\_count}$). |
| `+0x04` | 2 | `uint16_le` | **Name Length**: Byte length of the link group name ($\text{name\_len}$). |
| `+0x06` | $\text{name\_len}$ | `string` | **Group Name**: ASCII or UTF-8 display name (no trailing null). |
| `+var` | 2 | `uint16_le` | **Link Attributes**: Reserved/type flag (normally `0x0000`). |
| `+var+2` | 4 | `uint32_le` | **Room Count**: Number of rooms in this link group ($M$). |
| `+var+6` | $4 \times M$ | `uint32_le[]` | **Room IDs**: Array of 32-bit room identifiers referenced in `mcs.dat` / `DnData.dn`. |
| `+var+6+4M` | 10 | `bytes` | **Tail State**: 10-byte structure storing master room index, linked volume offsets, and mute sync flags. |

If `link_count == 0` (as in a default project without preset room links), `Link.dat` is exactly 8 bytes:
```hex
04 00 00 00 00 00 00 00
```

---

## 2. Event Timer Manager (`Event.dat`)

The event timer database manages RTC scheduled events, recurring calendar macros, and vacation mode activity.

### 2.1 File Header
### 2.2 Event Schedule Record

Each event schedule entry is exactly 62 bytes (`0x26 + 0x18`), consisting of a 38-byte configuration header followed by a 24-byte action & destination segment:

#### Header Segment (38 bytes / `0x26`):

| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | **Event Timer ID**: Unique 32-bit schedule identifier. |
| `+0x04` | 1 | `uint8` | **Vacation Mode Only**: `1` if this schedule only runs during vacation mode; `0` otherwise. |
| `+0x05` | 2 | `uint16_le` | **Start Month**: Calendar start month ($1\dots 12$). |
| `+0x07` | 2 | `uint16_le` | **Start Date**: Calendar start day ($1\dots 31$). |
| `+0x09` | 1 | `uint8` | **Repeat Type**: `0` = Once, `1` = Daily, `2` = Weekly, `3` = Monthly. |
| `+0x0A` | 2 | `uint16_le` | **Year**: Target year ($0$ for recurring). |
| `+0x0C` | 2 | `uint16_le` | **Month**: Target month ($0$ for recurring). |
| `+0x0E` | 2 | `uint16_le` | **Day**: Target day ($0$ for recurring). |
| `+0x10` | 7 | `uint8[7]` | **Days of Week**: 7 boolean flags for active days (index 0 = Sunday, 6 = Saturday). |
| `+0x17` | 1 | `uint8` | **On Macro Trigger Type**: `0` = Fixed Time, `1` = Sunrise Offset, `2` = Sunset Offset. |
| `+0x18` | 2 | `uint16_le` | **On Macro Hour**: Execution hour ($0\dots 23$) or signed sunrise/sunset offset. |
| `+0x1A` | 2 | `uint16_le` | **On Macro Minute**: Execution minute ($0\dots 59$). |
| `+0x1C` | 1 | `uint8` | **Off Macro Trigger Type**: `0` = None / Fixed Time, `1` = Sunrise, `2` = Sunset. |
| `+0x1D` | 2 | `uint16_le` | **Off Macro Hour**: Off execution hour ($0\dots 23$). |
| `+0x1F` | 2 | `uint16_le` | **Off Macro Minute**: Off execution minute ($0\dots 59$). |
| `+0x21` | 4 | `uint32_le` | **Event Macro ID**: Macro object identifier to execute in `DnData.dn`. |
| `+0x25` | 1 | `uint8` | **Reserved / Padding**: Single alignment byte (`0x00`). |

#### Action & Target MAC Segment (24 bytes / `0x18`):

In memory, this segment is loaded into object offset `+0x27` (`4 + 0x27 = 0x2b` aligned relative to structure start; 1 byte gap after 38 bytes at `+0x26`). On disk, it is written directly following the 38-byte header:

| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x26` | 16 | `bytes` | **Action Parameters**: Parameter block passed to the event macro executor (or all zeros). |
| `+0x36` | 8 | `bytes` | **Target Client / Station MAC**: 8-byte network identifier (EUI-48 MAC address padded with two zero bytes, or broadcast). Corresponds to in-memory offset `+0x37`. |
### 2.3 File Footer (2 bytes)

Following all $S$ schedule records, a 2-byte global status footer concludes the file:

| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 1 | `uint8` | **Vacation Mode Enabled**: Global vacation mode toggle (`1` = active, `0` = inactive). |
| `+0x01` | 1 | `uint8` | **Vacation Email Notification**: Flag enabling email alerts on vacation trigger (`1` = enabled, `0` = disabled). |

In a project with no scheduled events, `Event.dat` is exactly 11 bytes:
```hex
00 10 00 00 01 00 00 00 00 00 00
```

---

## 3. Alarm Object Table (`Alarm.dat`)

The alarm database manages scheduled audio alarms, wake-up music sources, and client chime notifications per room.

### 3.1 File Header

The file begins with a 9-byte header:

| Offset | Size | Type | Description |
|---|---|---|---|
| `0x00` | 4 | `uint32_le` | **Version**: File format version. Must be $\ge 2$; modern projects use `0x00003000` (Version 3.0). |
| `0x04` | 1 | `uint8` | **Format Flag**: Always `0x01`. |
| `0x05` | 4 | `uint32_le` | **Room Count**: Number of room alarm blocks ($R$). Must match total room count in project. |

### 3.2 Per-Room Alarm Record

For each of the $R$ rooms:

| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | **Room ID**: 32-bit room index ($0, 1, 2\dots$). |
| `+0x04` | 4 | `uint32_le` | **Alarm Count**: Number of alarm definitions configured in this room ($A$). |

### 3.3 Per-Alarm Record (Version $\ge$ 3)

When $A > 0$, $A$ variable-length alarm records follow:

| Offset | Size | Type | Description |
|---|---|---|---|
| `+0x00` | 4 | `uint32_le` | **Alarm ID**: Unique 32-bit alarm identifier. |
| `+0x04` | 2 | `uint16_le` | **Name Length**: Byte length of the alarm label ($L$). (Note: 4 bytes in Version < 3). |
| `+0x06` | $L$ | `string` | **Alarm Name**: Label string (e.g. "Morning Wakeup"). |
| `+0x06+L` | 1 | `uint8` | **Enabled**: `1` = active, `0` = disabled. |
| `+0x07+L` | 7 | `uint8[7]` | **Active Days**: 7 boolean flags for active days (Sun..Sat). |
| `+0x0E+L` | 2 | `uint16_le` | **Alarm Hour**: Trigger hour ($0\dots 23$). |
| `+0x10+L` | 2 | `uint16_le` | **Alarm Minute**: Trigger minute ($0\dots 59$). |
| `+0x12+L` | 2 | `uint16_le` | **Duration / Snooze**: Snooze timeout in minutes or total alarm duration. |
| `+0x14+L` | 16 | `bytes` | **Audio Source Data**: Input source ID, initial volume, volume ramp rate, and tone selector. |
| `+0x24+L` | 8 | `bytes` | **Target Device MAC**: MAC address of client UI / interface triggering the alarm. |
| `+0x2C+L` | 1 | `uint8` | **Alarm Type**: Type flags (radio, tone, playlist macro). |

In a 3-room project with no alarms configured, `Alarm.dat` is exactly 33 bytes:
```hex
00 30 00 00 01 03 00 00 00
00 00 00 00 00 00 00 00
01 00 00 00 00 00 00 00
02 00 00 00 00 00 00 00
```

---

## 4. Volume Feedback Descriptors (`FB/*.fb`)

Feedback descriptor files are stored under `/remote/Store/data/FB/` and packaged in `FB.tar`. They define channel feedback bindings for audio matrix and amplifier drivers.

### 4.1 File Naming & Indexing

Files are named `NNN.fb` using a 3-digit zero-padded index ($010\dots 033$):
- Indices `010` to `017`: Stereo Left/Right feedback for Zones 1 through 8.
- Indices `018` to `025`: Left Mono feedback for Zones 1 through 8.
- Indices `026` to `033`: Right Mono feedback for Zones 1 through 8.

### 4.2 Content Format

Each file contains a single plain ASCII string with **no trailing newline**:

```
ZONE=<zone_number><channel_mode>
```

#### Valid Formats:
- **Stereo LR:** `ZONE=1LR` through `ZONE=8LR` (8 bytes each).
- **Left Mono:** `ZONE=1L Mono` through `ZONE=8L Mono` (12 bytes each).
- **Right Mono:** `ZONE=1R Mono` through `ZONE=8R Mono` (12 bytes each).

The numeric index `NNN` corresponds to the zone mode / descriptor index referenced by multizone amplifier drivers and HDA feedback bindings in `DnData.dn`. When Two-Way drivers query volume feedback, `BS.exe` constructs the path `/remote/Store/data/FB/NNN.fb` and parses the zone channel descriptor.

---

## 5. Verification & Test Vectors

### 5.1 Test Vector: Empty Default System Tables
An initial project with 3 rooms (`room_id = 0, 1, 2`) and no links or timers:

- **`Link.dat` (8 bytes):**
  `0400000000000000`
- **`Event.dat` (11 bytes):**
  `0010000001000000000000`
- **`Alarm.dat` (33 bytes):**
  `003000000103000000000000000000000001000000000000000200000000000000`
- **`FB/010.fb` (8 bytes):**
  `5a4f4e453d314c52` (`ZONE=1LR`)

All test vectors re-serialize byte-identically with reference files extracted from controller storage.

---

## 6. Open Items
- The exact 16-byte payload format in `Alarm.dat` (`alarm_source_data`) will be cataloged alongside macro action parameters in `32-download-database` (`DnData.dn`).
