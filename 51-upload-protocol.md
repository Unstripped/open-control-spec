---
title: "Programming and Upload Protocol"
section: "51-upload-protocol"
version: "0.1.0"
status: review
circumvention: false
---
# Programming and Upload Protocol

## 1. Scope

A programmer deploys project data and triggers reconfiguration on an MRX controller through three networked services:

1. **File Transfer Service (TCP Port 21 / 22):** Files are staged into `/remote/Store/updateroot/` using standard FTP (`vsftpd`) or SFTP/SCP (`dropbear`).
2. **Staging Query Service (TCP Port 51520):** Staged file presence and checksum tokens are queried and validated via length-prefixed request/response frames.
3. **Reload Trigger Listener (UDP Port 61520):** A binary datagram instructs the controller's runtime daemon to ingest staged files and reboot.

Confidence tags follow Section 00 §4.5 (`[CONFIRMED]`, `[STATIC]`, `[INFERRED]`).

---

## 2. Protocol State Machine

```
   +---------------------+
   | 1. Disconnected     |
   +----------+----------+
              |
              | Connect TCP Port 21 (FTP) or Port 22 (SFTP)
              v
   +---------------------+
   | 2. File Staging     | Transfer staged files & manifests to updateroot/
   +----------+----------+
              |
              | Connect TCP Port 51520 (rmslocalsvrd)
              v
   +---------------------+
   | 3. Query & Verify   | 0x2010 (GET_VERSION) & 0x2020 (GET_CHECKSUM)
   +----------+----------+
              |
              | Transmit UDP Datagram to Port 61520
              v
   +---------------------+
   | 4. Reload Trigger   | Opcode 0x2030 (RLS_NEW_VERSION)
   +----------+----------+
              |
              | Controller verifies manifests, unpacks files, updates version.info
              v
   +---------------------+
   | 5. Reboot & Run     | System reboots into updated configuration
   +---------------------+
```

---

## 3. Staging Query Service (TCP Port 51520)

### 3.1 Framing

All frames exchanged on TCP port 51520 use 16-bit little-endian length prefixing:

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 2 | `length` | uint16 LE | Total size of subsequent payload bytes |
| 0x02 | `length` | `payload` | bytes | Frame payload |

### 3.2 Frame Header

Every payload begins with a 20-byte standard header:

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `sequence` | uint32 LE | Transaction sequence number |
| 0x04 | 4 | `model_id` | uint32 LE | Hardware model identifier (`0x8008` for MRX-15/MRX-10) |
| 0x08 | 8 | `client_id` | bytes | 6-byte target MAC address followed by `0x0000` |
| 0x10 | 4 | `command` | uint32 LE | Command opcode |

### 3.3 Command `0x2010` (`GET_VERSION`)

- **Direction:** Programmer → Controller.
- **Request Body:** None (header only, payload length = 20 bytes, total frame length = 22 bytes).
- **Response Command:** `0x2011`.
- **Response Body:**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `count` | uint32 LE | Total number of staged files matching model/client prefix |
| 0x04 | var | `root` | string | Staging directory root terminated by TAB (`/updateroot\t`) |
| var | var | `entries` | array | `count` file records: `<filename>\t` (string) + `token` (uint32 LE) |

### 3.4 Command `0x2020` (`GET_CHECKSUM`)

- **Direction:** Programmer → Controller.
- **Request Body:**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `count` | uint32 LE | Number of files requested |
| 0x04 | var | `filenames` | string | TAB-separated list of staged filenames terminated by `\0` |

- **Response Command:** `0x2021`.
- **Response Body:**

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0x00 | 4 | `count` | uint32 LE | Number of checksum values returned |
| 0x04 | `count` × 4 | `checksums` | uint32 LE array | Rotating byte-sum checksum per file (`0xFFFFFFFF` if absent/unreadable) |

---

## 4. Reload Trigger Service (UDP Port 61520)

### 4.1 Datagram Format

The reload trigger is a 28-byte UDP datagram sent to port 61520:

| Offset | Size | Field | Type | Allowed Values | Description |
|---|---|---|---|---|---|
| 0x00 | 2 | `length` | uint16 LE | `0x001C` (28) | Total datagram size in bytes `[STATIC]` |
| 0x02 | 4 | `sequence` | uint32 LE | Non-zero integer | Sequence counter `[STATIC]` |
| 0x06 | 4 | `model_id` | uint32 LE | `0x00008008` | Target hardware model ID `[CONFIRMED]` |
| 0x0A | 6 | `dest_mac` | 6 bytes | Target MAC or `FF:FF:FF:FF:FF:FF` | Controller MAC address filter `[CONFIRMED]` |
| 0x10 | 2 | `reserved` | uint16 LE | `0x0000` | Reserved / Sub-ID padding `[STATIC]` |
| 0x12 | 4 | `command` | uint32 LE | `0x00002030` | Opcode `RLS_NEW_VERSION` `[CONFIRMED]` |
| 0x16 | 6 | `sender_mac` | 6 bytes | `00:00:00:00:00:00` or Master MAC | Originating programmer / master MAC `[CONFIRMED]` |

### 4.2 Ingestion Semantics

Upon receipt of a valid `RLS_NEW_VERSION` datagram:
1. `BS.exe` verifies `model_id` matches `0x8008` and `dest_mac` matches local hardware MAC or broadcast `[CONFIRMED]`.
2. Non-empty manifests matching `M8008*_M<MAC>*.info` and `M8008*_A*.info` must be present in `/remote/Store/updateroot/` `[CONFIRMED]`.
3. The installer scans staged files and performs installation based on original filename patterns:
   - Files whose staged names already appear in `/remote/Store/Prg/version.info` are skipped `[CONFIRMED]`.
   - Staged names containing `_DnData` are installed to `/remote/Store/data/DnData.dn` `[CONFIRMED]`.
   - Staged names containing `_rms.ini` are installed to `/remote/Store/data/rms.ini` `[CONFIRMED]`.
   - Staged names containing `_edit.dat.gz` are copied to `/remote/Store/CCS/edit.dat.gz` `[CONFIRMED]`.
   - Staged names containing `_FB` or `_TwoWay` are untarred into `/remote/Store/data/` `[STATIC]`.
   - Staged names containing `_BS.exe`, `_tclpkg`, `_subproc*`, or `_ZigbeePRO` are copied into `/remote/Store/Prg/` `[STATIC]`.
   - **Ignored Files:** Staged files named `mcs.dat`, `mcs_info.dat`, `Link.dat`, `Event.dat`, `Alarm.dat`, and `position.info` are never installed by the firmware staging installer `[CONFIRMED]`. They must be provisioned directly in `/remote/Store/data/`.
4. The installer creates `/remote/Store/Prg/version.info` by concatenating the staged `A` and `M` manifests (`cat A M > /remote/Store/Prg/version.info`) `[CONFIRMED]`.
5. The runtime daemon terminates DHCP (`killall udhcpc`), syncs filesystems, and reboots the controller (`sync; sleep 2; reboot`) `[CONFIRMED]`.

### 4.3 Alternative Activation Paths

1. **Boot-Time Staging Installer:** At every system startup, `BS.exe` checks for staged files before loading the database. If new staged files exist in `/remote/Store/updateroot/` and are not yet in `version.info`, `BS.exe` executes the exact same installer logic, creates `version.info`, logs `Update success`, and reboots automatically `[CONFIRMED]`.
2. **Network Administrative Reboot (NET_REBOOT):** TCP port 51540 hosts an unauthenticated diagnostic server (`McsServerThread`). A 9-byte packet carrying command opcode `0x0476` causes `BS.exe` to return an 11-byte diagnostic response and immediately execute `system("sync; sleep 2; reboot")` without requiring SSH credentials `[CONFIRMED]`.

## 5. Error Behavior

1. **Short / Malformed Frame on TCP 51520:** The server terminates the connection or returns `count = 0`.
2. **Missing Files in `GET_CHECKSUM`:** Returns `0xFFFFFFFF` for any missing or unreadable filename.
3. **Invalid MAC or Model in UDP 61520:** Silently ignored by `BS.exe`.
4. **Missing Manifests during Reload:** If `M8008_M<MAC>*.info` or `M8008_A*.info` are absent or unreadable, `BS.exe` aborts ingestion and continues normal execution without rebooting.

---

## 6. Test Vectors

### 6.1 `GET_VERSION` Request Frame (Port 51520)

Hex representation (22 bytes total):
```
14 00 01 00 00 00 08 80 00 00 00 1f b8 00 11 22 00 00 10 20 00 00
```
- Length: `0x0014` (20 bytes payload).
- Seq: `1`, Model: `0x8008`.
- Client ID: `00:1f:b8:00:11:22:00:00`.
- Command: `0x2010` (`GET_VERSION`).

### 6.2 `RLS_NEW_VERSION` Trigger Datagram (Port 61520)

Hex representation (28 bytes total):
```
1c 00 01 00 00 00 08 80 00 00 00 1f b8 00 11 22 00 00 30 20 00 00 00 00 00 00 00 00
```
- Length: `0x001C` (28 bytes).
- Model: `0x00008008`.
- Destination MAC: `00:1f:b8:00:11:22`.
- Reserved: `0x0000`.
- Command: `0x00002030` (`RLS_NEW_VERSION`).
- Sender MAC: `00:00:00:00:00:00`.

---

## 7. Open Items

1. **SNP Audio Transfer:** Streaming Network Player download synchronization (`0x2060` / `0x2061`) is optional for standard controller programming.
2. **Runtime Configuration Provisioning:** Because `mcs.dat`, `mcs_info.dat`, and device link tables are bypassed by the firmware staging installer, third-party programmers must provision them directly into `/remote/Store/data/`.
