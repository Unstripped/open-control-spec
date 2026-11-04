---
title: "Clean-Room Implementer Brief & Architecture Guide"
section: "IMPLEMENTER"
version: "0.1.0"
status: ratified
circumvention: true
---

# Clean-Room Implementer Brief & Architecture Guide

> **LEGAL NOTICE & NOMINATIVE FAIR USE DISCLAIMER:**  
> This document is part of an independent, clean-room technical specification authored solely to achieve device and network interoperability. This project is not affiliated with, sponsored by, or endorsed by Universal Remote Control, Inc. (URC). All product names, trademarks, and registered trademarks (including "URC", "Total Control", "MRX-8", "MRX-10", "MRX-15", "MRX-30", and "Accelerator") are the property of their respective owners, and are used here exclusively for nominative identification and technical compatibility purposes under 15 U.S.C. § 1125 and controlling fair use doctrines.

## 1. Executive Summary & Mission

This document provides the foundational engineering brief for an independent, clean-room software team or agent building an open-source programming suite for **URC Total Control MRX network controllers** (e.g., MRX-15, MRX-30, MRX-10, MRX-8).
### 1.1 The Objective
The objective is to produce a self-contained, dealer-independent programming and deployment toolchain comprising:
1. **A Human-Friendly Project Definition Format:** A declarative schema (e.g., YAML or TOML) describing rooms, controlled devices, macro logic, state variables, and user interface layouts.
2. **An MRX Project Compiler:** A compilation engine that transforms the high-level project definition into the exact bundle of binary databases, topology JSONs, automation tables, and upload manifests required by the controller's runtime daemons.
3. **A LAN Deployment Tool:** An automated network utility that discovers MRX controllers on the local network, stages the compiled project archive via standard file transfer protocols, issues the software reload trigger, and verifies operational health following activation.
4. **A Runtime Control & Diagnostic Client:** A network client implementing the controller's bidirectional control protocol to execute macros, track state variables, and monitor controller events.

### 1.2 The Clean-Room Contract
This specification suite (`spec/`) represents the **sole technical authority** for the clean-room implementer.
- **Strict Black-Box Boundary:** The implementer has access **only** to the documents in `spec/`. The implementer does not possess, inspect, or reference vendor firmware dumps, decompiler outputs, disassembly listings, or vendor proprietary software.
- **Zero Proprietary Key Distribution:** The clean-room programmer distributes zero proprietary encryption keys or certificates. Where encryption is required for third-party module packaging or client licensing, the toolchain relies on clean user configuration injection.
- **Autonomous LAN Operation:** The target hardware is completely self-sufficient. Controller runtime daemons execute automation logic locally on the LAN without requiring internet connectivity, cloud authentication, or vendor dealer maintenance portals.

---

## 2. End-to-End System Architecture

The following diagram illustrates the complete data flow and execution lifecycle across the clean-room programmer toolchain and the physical MRX controller:

```
+-----------------------------------------------------------------------------------+
|                           CLEAN-ROOM PROGRAMMER TOOLCHAIN                         |
+-----------------------------------------------------------------------------------+
|                                                                                   |
|   +---------------------------------------------------------------------------+   |
|   | Human Project Definition (project.yaml / project.toml)                     |   |
|   | - Controller identity (model, MAC, static/DHCP IP)                        |   |
|   | - Rooms & Subsystems (living room, theater, patio)                        |   |
|   | - Devices & Transports (IR codes, RS-232 serial, IP endpoints, relays)    |   |
|   | - Macros & State Variables (power toggles, source switching, delays)      |   |
|   | - Mobile & Remote Client Seats (iOS, Android, native touchscreens)        |   |
|   +---------------------------------------------------------------------------+   |
|                                         |                                         |
|                                         v                                         |
|   +---------------------------------------------------------------------------+   |
|   |                          MRX Project Compiler                             |   |
|   |                                                                           |   |
|   |   1. Topology Model Generator:                                            |   |
|   |      -> mcs_info.dat  (Base station, client seats, device inventory)      |   |
|   |      -> mcs.dat       (Matrix zones, device functions, audio bindings)    |   |
|   |      (Section 30: 30-project-model.md)                                    |   |
|   |                                                                           |   |
|   |   2. Controller Binary Database Compiler:                                 |   |
|   |      -> DnData.dn     (56-byte header, 26-table directory, inner records) |   |
|   |      (Section 32: 32-download-database.md)                                |   |
|   |                                                                           |   |
|   |   3. Automation & Schedule Tables:                                        |   |
|   |      -> Link.dat, Event.dat, Alarm.dat, FB/*.fb                           |   |
|   |      (Section 31: 31-automation-tables.md)                                |   |
|   |                                                                           |   |
|   |   4. Dealer Backup Archive (Minimal / Stub):                              |   |
|   |      -> CCS/edit.dat.gz                                                   |   |
|   |      (Section 33: 33-master-project-database.md)                          |   |
|   |                                                                           |   |
   |   5. Checksum & Manifest Staging Engine:                                  |   |
   |      -> 32-bit rotating byte-sum checksum token calculation               |   |
   |      -> M8008_A*.info and M8008_M*.info manifests                         |   |
   |      -> Staged filenames (e.g., M8008_M<MAC>0000_C<token>_DnData.dn)      |   |
   |      (Section 20: 20-upload-manifest.md & Section 51: 51-upload-protocol) |   |
   +---------------------------------------------------------------------------+   |
                                         |                                         |
                                         v                                         |
|   +---------------------------------------------------------------------------+   |
|   |                     Compiled Release & Staging Set                        |   |
|   |   1. Direct Provisioning to /remote/Store/data/:                          |   |
|   |      ├── mcs.dat, mcs_info.dat                                            |   |
|   |      └── Link.dat, Event.dat, Alarm.dat, FB/*.fb                          |   |
|   |   2. Staged Release Package in /remote/Store/updateroot/:                 |   |
|   |      ├── M8008_A0000000000000000_T<time>.info (model manifest)           |   |
|   |      ├── M8008_M<MAC>0000_T<time>.info        (device manifest)          |   |
|   |      ├── M8008_M<MAC>0000_C<token>_DnData.dn  (database)                 |   |
|   |      ├── M8008_M<MAC>0000_C<token>_edit.dat.gz (dealer recovery)         |   |
|   |      └── rms.ini                               (service flags)            |   |
|   +---------------------------------------------------------------------------+   |
                                         |                                         |
                                         v                                         |
|   +---------------------------------------------------------------------------+   |
|   |                         LAN Deployment Engine                             |   |
|   |   1. Discovery Probe: UDP 61510 (0xC350 PACKET_FIND)                      |   |
|   |      -> Receive 0xC353 PACKET_FIND_ACK (IP, MAC, Model, Version)          |   |
|   |      (Section 50: 50-discovery.md)                                        |   |
|   |                                                                           |   |
|   |   2. Staging Transfer: FTP Port 21 (anonymous) or SFTP Port 22            |   |
|   |      -> Store staged release files into /remote/Store/updateroot/         |   |
|   |      (Section 51: 51-upload-protocol.md)                                  |   |
|   |                                                                           |   |
|   |   3. Reload Trigger: UDP Port 61520                                       |   |
|   |      -> Send 28-byte datagram with opcode 0x2030 (RLS_NEW_VERSION)        |   |
|   |      (Section 51: 51-upload-protocol.md)                                  |   |
|   +---------------------------------------------------------------------------+   |
|                                                                                   |
+-----------------------------------------------------------------------------------+
                                          |
                                          | Local Network (LAN)
                                          v
+-----------------------------------------------------------------------------------+
|                        PHYSICAL MRX CONTROLLER RUNTIME                            |
+-----------------------------------------------------------------------------------+
|                                                                                   |
|   +---------------------------------------------------------------------------+   |
|   | Supervisor & Staging Daemons                                              |   |
|   | - rmslocalsvrd.exe (TCP 51520): Checksum & staging verification           |   |
|   | - CheckThread in BS.exe (UDP 61520): Ingests release datagram             |   |
|   |   -> Validates updateroot tokens against version.info                     |   |
|   |   -> Atomically moves / unpacks files into /remote/Store/data & /Prg      |   |
|   |   -> Executes /sbin/reboot                                                |   |
|   +---------------------------------------------------------------------------+   |
|                                         |                                         |
|                                         v Reboot & Ingestion (~25-30s)            |
|   +---------------------------------------------------------------------------+   |
|   | Active Controller Runtime                                                 |   |
|   | - BS.exe (TCP 51510): Master control daemon, macro engine, state tables   |   |
|   | - rui_svrd.exe (TCP 51530): UI rendering engine & Two-Way Tcl runtime     |   |
|   | - provisioningd (UDP 61510): Network discovery broadcast listener         |   |
|   | - Hardware I/O: Infrared emitters, RS-232 serial, relays, 12V sensors     |   |
|   +---------------------------------------------------------------------------+   |
|                                                                                   |
+-----------------------------------------------------------------------------------+
                                          ^
                                          |
+-----------------------------------------------------------------------------------+
|                     VERIFICATION & CONTROL DIAGNOSTIC CLIENT                      |
+-----------------------------------------------------------------------------------+
|   - Connects to TCP 51510 (0x02 framing)                                          |
|   - Query Status: Opcode 0x0FE1 (GET_NET_STATUS)                                  |
|   - Synchronize Variables: Opcode 0x089A (VARIABLE_CMD_GET_TABLE)                 |
|   - Trigger Macro Execution: Opcode 0x07DA (MACRO_ACTION) -> ACK 0x07DB           |
|   (Section 52: 52-control-protocol.md & Section 53: 53-client-protocols.md)       |
+-----------------------------------------------------------------------------------+
```

---

## 3. High-Level Project Definition (YAML / TOML Schema)

The clean-room programmer uses a human-readable, declarative project specification that abstracts away binary offsets, rotating checksums, and table indices. Below is the canonical structure for an MRX project configuration:

```yaml
# mrx-project.yaml - Declarative Clean-Room Configuration Schema

project:
  name: "Residence Automation"
  schema_version: "1.0.0"
  timezone: "America/New_York"

controller:
  model: "MRX-15"                    # Target hardware model (MRX-15, MRX-30, MRX-10, MRX-8)
  mac_address: "00:1F:B8:00:11:22"   # Hardware Ethernet MAC address (required for manifests)
  network:
    dhcp: true                       # Or static IP configuration
    # ip: "192.168.1.100"
    # gateway: "192.168.1.1"

# State variables accessible across rooms, macros, and modules
variables:
  - id: 1
    name: "System_Power_State"
    type: "boolean"                  # boolean (0/1), integer, or string
    initial_value: 0
  - id: 2
    name: "Active_Audio_Zone"
    type: "integer"
    initial_value: 0

# Client user interface seats (mobile phones, touchscreens, remotes)
clients:
  - id: 5
    name: "Living Room iPhone"
    model: "IPhoneV2"                # Client profile (Section 30 & 53)
    mac_address: "00:00:00:00:00:00"

# Controlled hardware devices (AV gear, TVs, lighting, relays)
devices:
  - id: 3
    name: "Apple TV"
    category: "AUX"
    transport: "IR"                  # IR, SERIAL, IP, RELAY
    port: 1                          # Physical IR emitter port 1
    commands:
      - name: "Power"
        code: "0000 006d 0022 0002 ..." # Hexadecimal Pronto / pulse timing sequence
      - name: "Menu"
        code: "0000 006d 0022 0002 ..."

  - id: 4
    name: "AV Receiver"
    category: "AVR"
    transport: "IP"
    endpoint: "192.168.1.50:23"
    commands:
      - name: "PowerOn"
        string: "PWON\r\n"
      - name: "VolumeUp"
        string: "MVUP\r\n"

# Room definitions, zone audio routing, and macro sequences
rooms:
  - id: 0
    name: "Living Room"
    device_bindings: [3, 4]          # Associated devices available in room

    macros:
      - id: 0
        name: "Watch Apple TV"
        actions:
          - type: "variable_set"
            variable_id: 1
            value: 1
          - type: "device_command"
            device_id: 4
            command: "PowerOn"
          - type: "delay"
            duration_ms: 1000
          - type: "device_command"
            device_id: 3
            command: "Menu"

      - id: 1
        name: "Room Off"
        actions:
          - type: "variable_set"
            variable_id: 1
            value: 0
          - type: "device_command"
            device_id: 4
            command: "PowerOff"
```

---

## 4. Compiler Subsystem Architecture

The **MRX Project Compiler** transforms the human-readable project specification into target artifacts adhering strictly to the binary layouts and schemas defined across `spec/`.

### 4.1 Target Artifact Inventory
The compiler produces eight primary files that constitute a self-consistent program package:

| Target File | Format | Spec Reference | Functional Role |
|---|---|---|---|
| `mcs_info.dat` | JSON (compact) | `spec/30-project-model.md` | Catalog of Base Stations, client seats, and controlled device descriptors. |
| `mcs.dat` | JSON (compact) | `spec/30-project-model.md` | Zone matrix routing, device function indices, and audio endpoint links. |
| `DnData.dn` | Binary Database | `spec/32-download-database.md` | Master execution database containing 26 tables, IR/serial codes, macros, and room bindings. |
| `Link.dat` | Binary Table | `spec/31-automation-tables.md` | Cross-variable bindings and conditional logic links. |
| `Event.dat` | Binary Table | `spec/31-automation-tables.md` | Sensor triggers, relay state events, and hardware interrupts. |
| `Alarm.dat` | Binary Table | `spec/31-automation-tables.md` | Scheduled RTC events, recurring timers, and automated actions. |
| `CCS/edit.dat.gz` | Gzip Archive | `spec/33-master-project-database.md` | Preserved master project database (dealer upload recovery). |
| `rms.ini` | INI Config | `spec/90-examples/minimal-program.md` | Remote management service disable/enable flags (22 bytes). |

### 4.2 Cross-Reference Integrity Rules
When compiling these files, the compiler MUST enforce strict relational integrity:
1. **Object ID (`obj_id`) Consistency:** Every `obj_id` assigned to a Base Station, Client, Device, Room, or Variable MUST match identically across `mcs_info.dat`, `mcs.dat`, and `DnData.dn`.
2. **Table 4 Client Licensing (`ClientTable`):** Any mobile client seat defined in `mcs_info.dat` (`clients` array) MUST have a corresponding 160-byte authorization record in Table 4 of `DnData.dn`. The `sub_type` MUST be `1` (Authorized), and the seat type MUST be `0x0001` (iOS Mobile) or `0x0002` (Android Mobile).
3. **Table 5 Subdevice Routing:** Connected devices in Table 5 of `DnData.dn` MUST match the transport definition (`target_type == 1` for IR/Serial, `target_type == 3` for IP).
4. **Room Power Macros:** In Table 10 of `DnData.dn` (`RoomObjTable`), each room entry MUST contain two sequential parameterless `ButtonObj` entries representing Room Power On and Room Power Off macro sequences.

### 4.3 Staging Manifests, Checksums & Delivery Paths
After generating the runtime files, the programmer prepares them for controller deployment following the rules in `spec/20-upload-manifest.md` and `spec/51-upload-protocol.md`:

1. **Content Checksum Calculation:**
   For every staged content file, compute the 32-bit rotating byte-sum checksum token specified in `spec/20-upload-manifest.md` §3:
   $$\text{term}(i) = (b[i] \ll (i \bmod 8)) \mid (b[i] \gg (8 - (i \bmod 8)))$$
   $$\text{Checksum} = \left(\sum_{i=0}^{n-1} \text{term}(i)\right) \bmod 2^{32}$$
   where each term is evaluated in at least 16-bit precision (with $\text{term}(i) = b[i]$ when $i \bmod 8 = 0$). In staged filenames, this renders as 8 uppercase zero-padded hex digits prefixed by `C` (e.g., `C085E1F22`), while in manifest columns it renders in uppercase hex without leading zeros.

2. **Staged Filename Syntax:**
   Files staged in `/remote/Store/updateroot/` adhere to the 33-character fixed prefix grammar in `spec/20-upload-manifest.md` §2:
   `<prefix: 5 chars>_<address: 17 chars>_<kind: 1 char><token: 8 hex digits>_<original_filename>`
   - `prefix`: 1-letter class + 4-hex model ID (e.g., `M8008` for MRX-15).
   - `address`: 1-letter address class + 16-hex address ID (`A0000000000000000` for all devices of that model; `M<12-hex MAC>0000` for a specific physical controller).
   - `kind`: `C` (content checksum) for content files, `T` (Unix timestamp) for manifests and position records.
   - Examples:
    - Binary database: `M8008_M001FB80011220000_C085E1F22_DnData.dn`
    - Master recovery: `M8008_M001FB80011220000_C5C1F1BD_edit.dat.gz`
    - Device manifest: `M8008_M001FB80011220000_T632B7C9D.info`
     - Model manifest: `M8008_A0000000000000000_T632B7C9D.info`

3. **Staging vs. Direct Provisioning Delivery Paths:**
   As specified in `spec/51-upload-protocol.md` §4.2 and §7, the controller firmware reload installer (`BS.exe` installer thread) handles a specific set of staged files in `/remote/Store/updateroot/`:
   - `_DnData` files are installed to `/remote/Store/data/DnData.dn`.
   - `_rms.ini` files are installed to `/remote/Store/data/rms.ini`.
   - `_edit.dat.gz` files are installed to `/remote/Store/CCS/edit.dat.gz`.
   - `_FB` and `_TwoWay` tarballs are unpacked into `/remote/Store/data/`.
   - Manifests are concatenated into `/remote/Store/Prg/version.info`.
   - **Direct Provisioning:** Staged files named `mcs.dat`, `mcs_info.dat`, `Link.dat`, `Event.dat`, and `Alarm.dat` are bypassed by the firmware staging installer. Programmers MUST provision these runtime topology and automation files directly into `/remote/Store/data/` via FTP or SFTP.


### 4.4 Cryptographic Requirements & Key Boundaries
The specification suite categorizes subsystems into distinct cryptographic boundaries:

1. **Keyless Subsystems (Zero Keys Required):**
   - **Network Discovery (`spec/50-discovery.md`):** UDP 61510 probe (`0xC350`) and response (`0xC353`) frames are plaintext.
   - **Deployment & Reload Trigger (`spec/51-upload-protocol.md`):** Anonymous FTP staging and the 28-byte UDP 61520 reload datagram require only the controller's MAC address and model ID.
   - **Control Protocol Wire Frames (`spec/52-control-protocol.md`):** TCP 51510 status queries (`0x0FE1`), variable synchronization (`0x089A`), and macro triggers (`0x07DA`) are unencrypted.
   - **IR, Relay, Sensor & Variable Automation:** Physical IR emitter assignments (port index), dry contact relays, 12V voltage sensor triggers, room navigation, and state variables serialize without encryption in `DnData.dn` Table 5, Table 8, and Table 10. The **Minimal Program** (`spec/90-examples/minimal-program.md`) compiles and runs end-to-end completely keyless.

2. **Subsystems Requiring the Universal Firmware Key (`content-aes-static`):**
   - **RS-232 Serial & Direct IP Endpoints (`spec/32-download-database.md` §4.2, §5.3):** In `DnData.dn` Table 5, when a connected device uses transport `2` (Serial) or `3` (Network/IP), the target IP/port or serial baud parameters are sealed in an 8-byte framing prefix (`org_size`, `enc_size`) followed by AES-128 ECB ciphertext. The controller daemon decrypts this at runtime using the static firmware key.
   - **Client Seat Activation & Mobile Profiles (`spec/32-download-database.md` §4.2, §5.2):** In Table 4 (`ClientTable`), mobile client records carry 16-byte encrypted client activation descriptors.
   - **Two-Way Tcl Driver Modules (`spec/21-containers.md` §3–4, `spec/32-download-database.md` §4.1):** Table 0 (`ConfigObjTable`) contains the 16-byte project symmetric key wrapped under the static firmware key. The controller decrypts Table 0 to reveal the project key, which it uses to decrypt Two-Way `.tcl` scripts (`ENC DATA` format: AES-128 ECB + HMAC-SHA256) into `/tmp/` at boot.

3. **Mandatory External Key Provisioning Contract:**
   - Clean-room programmer source code, compilers, and binaries **MUST NOT** embed, bundle, hardcode, or automatically download proprietary vendor keys, private certificates, or seed vectors.
   - Compilers and deployment utilities **MUST** enforce runtime external key injection:
     - Users compiling encrypted structures (such as direct IP/Serial device profiles, mobile client seats, or Two-Way Tcl modules) MUST supply the necessary key at runtime via an explicit command-line flag (e.g., `--firmware-key /path/to/key.bin`) or an environment variable (e.g., `URC_FIRMWARE_KEY`).
     - When key material is absent from the runtime environment, the compiler MUST abort gracefully with an informative diagnostic error directing the user to supply their own key configuration, rather than falling back to bundled or remote secrets.

## 5. Deployment Subsystem Architecture

The **MRX LAN Deployment Engine** handles physical controller discovery, file transfer, and release activation.

### 5.1 Stage 1: Controller Discovery
- **Protocol:** UDP port 61510 broadcast (`spec/50-discovery.md`).
- **Discovery Packet:** The deployer sends a 20-byte `PACKET_FIND` datagram (`0xC350`).
- **Response Parsing:** The controller replies with an 88-byte `PACKET_FIND_ACK` datagram (`0xC353`) containing the controller's model identifier, hardware MAC address, firmware version, and IP address.

### 5.2 Stage 2: File Transfer & Staging
- **Protocols:** Standard FTP (TCP port 21, anonymous login `anonymous`/blank) or SFTP (TCP port 22) (`spec/51-upload-protocol.md`).
- **Transfer Sequence:**
  1. **Direct Provisioning:** Upload runtime configuration models (`mcs.dat`, `mcs_info.dat`, `Link.dat`, `Event.dat`, `Alarm.dat`, `FB/*.fb`) directly to `/remote/Store/data/`.
  2. **Release Staging:** Upload staged release files (`M8008_M*_DnData.dn`, `M8008_M*_edit.dat.gz`, `rms.ini`) and manifests (`M8008_A*.info`, `M8008_M*.info`) to `/remote/Store/updateroot/`.
- **Optional Staging Verification:** The deployer MAY connect to TCP port 51520 (`rmslocalsvrd.exe`) and issue opcode `0x2020` (`GET_CHECKSUM`) to verify that the controller computes the expected token for staged files before triggering reload (`spec/51-upload-protocol.md` §3.4).

### 5.3 Stage 3: Activation Trigger
- **Protocol:** UDP port 61520 unicast to the controller IP (`spec/51-upload-protocol.md` §4).
- **Datagram Layout:** A 28-byte datagram sent to `CheckThread` in `BS.exe`. All multi-byte integers are Little-Endian (`uint16_le`, `uint32_le`):
  - `0x00`: Datagram length (`uint16_le = 0x001C` [28 bytes])
  - `0x02`: Sequence ID (`uint32_le`)
  - `0x06`: Hardware Model ID (`uint32_le = 0x00008008` for MRX-15, bytes `08 80 00 00`)
  - `0x0A`: Destination MAC address (6 bytes, target controller hardware MAC or broadcast)
  - `0x10`: Reserved (`uint16_le = 0x0000`)
  - `0x12`: Command opcode (`uint32_le = 0x00002030` for `RLS_NEW_VERSION`, bytes `30 20 00 00`)
  - `0x16`: Sender MAC address (6 bytes, originating programmer MAC or zeros)
- **Controller Execution:** Upon receiving `0x2030`, `BS.exe` verifies the staged manifests against `/remote/Store/updateroot/`, installs the release files, creates `/remote/Store/Prg/version.info`, and triggers `/sbin/reboot`.
- **Downtime & Reconnection:** The controller reboots in approximately 25 to 35 seconds. The deployer polls via discovery (`0xC350`) or TCP port 51510 to confirm operational return.
---

## 6. Control & Diagnostic Client Architecture

The clean-room toolchain includes a diagnostic and runtime client to verify programs immediately after deployment without requiring a physical handheld remote.

### 6.1 Control Protocol (TCP Port 51510)
- **Framing:** 5-byte header prefix (`spec/52-control-protocol.md`):
  `0x02` + `uint16_le payload_len` + `uint16_le opcode` + `payload`
- **Core Operations:**
  - **Status Query (`0x0FE1`):** Requests controller network configuration, uptime, and operational mode. Returns `0x0FE2` ACK.
  - **Variable Synchronization (`0x089A`):** Retrieves the current values of all state variables in Table 11 (`VarTable`).
  - **Macro Trigger (`0x07DA`):** Executes any macro by passing `room_id`, `device_id`, `macro_id`, and `flags = 0x0001`. The controller responds with `0x07DB` ACK and executes the action sequence on hardware.
  - **Room Power Off (`0x000D`):** Triggers the automated room shutdown sequence.

### 6.2 Mobile App Client Emulation & Pairing
- **Mobile Pairing Lifecycle (`spec/53-client-protocols.md`):**
  1. **Pre-Registration:** Client connects to TCP port 51520 and sends `0x2070` with its client seat ID and IP address.
  2. **Model Retrieval:** Client connects to FTP port 21 and downloads `core_model.json`, `model.json`, `position.info`, and `ui_button_table.json`.
  3. **Session Authentication:** Client connects to TCP port 51510 and transmits `0x1389` (`CLIENT_LOGIN_REQ`) with seat ID and client MAC. Controller validates against Table 4 of `DnData.dn` and responds with `0x138C` (`CLIENT_LOGIN_ACK`).

---

## 7. Explicit Non-Goals & Scope Boundaries

To maintain engineering focus and strict legal compliance, the following items are **explicitly out of scope**:

1. **Firmware & OS Replacement:** The clean-room project does NOT replace the controller's Linux OS, U-Boot bootloader, or root filesystem. The programmer operates purely within the intended userland programming and staging interface.
2. **FPGA Bitstream Modification:** The toolchain does NOT reflash or modify the Altera Cyclone IV FPGA image (`MRX_30.rbf`). Hardware physical I/O (IR modulation, UART FIFOs, relays) is managed via the stock daemon interface.
3. **Cloud Portal & RMS Emulation:** Off-LAN dealer remote management via the vendor's SIP cloud tunnel (`urcrms.com`) is out of scope. The clean-room suite focuses exclusively on direct, local LAN control.
4. **Decompilation of Vendor PC Software:** The implementer MUST NOT decompile or reverse-engineer the vendor's Windows software (Accelerator 3, Total Control Flex). All compiler schemas and network protocols are derived strictly from `spec/`.
5. **Redistribution of Proprietary Assets:** The clean-room project does NOT bundle copyrighted vendor assets, including proprietary IR code databases, serial driver libraries, UI graphics, or font files. Users supply their own device codes or import open IR formats (e.g., Pronto Hex, Global Caché).

---

## 8. Canonical Reference: The Minimal Program

Before attempting complex multi-room or multi-device configurations, every implementer MUST build and verify the **Minimal Program Worked Example** documented in:
[`spec/90-examples/minimal-program.md`](90-examples/minimal-program.md)

### 8.1 The "Hello World" Test
The minimal program defines:
- Controller: MRX-15 (`00:1F:B8:00:11:22`)
- Room 0: "Living Room"
- Device 3: "Apple TV" (IR port 1)
- Variable 1: `TestVar` (boolean, initially 0)
- Macro 0: Sets `TestVar = 1`, fires IR command, delays 1000 ms, resets `TestVar = 0`.

### 8.2 Pass Criteria
An implementer passes the initial conformance milestone when their compiler can:
1. Synthesize all 8 minimal program files from a configuration file.
2. Successfully stage the files in `/remote/Store/updateroot/` on a physical or emulated controller.
3. Trigger the `0x2030` reload datagram and observe a clean controller reboot.
4. Connect to TCP port 51510, issue `0x07DA` to trigger Macro 0, and verify over TCP 51510 that `TestVar` toggles `0 -> 1 -> 0`.

---

## 9. Clean-Room Q&A Process (Filing Questions)

If an implementer encounters an ambiguity, contradiction, missing field definition, or unexpected controller error while developing against `spec/`, they MUST file an inquiry in:
[`spec/questions.md`](questions.md)

### 9.1 Inquiry Rules
- **Grounded in Spec Text:** Every question MUST reference a specific `spec/*.md` document, section, or table.
- **Black-Box Framing Only:** Inquiries MUST describe observable behavior, wire captures, packet layouts, or error responses.
- **Prohibited Inquiries:** Implementers MUST NOT ask:
  - *"How does the vendor decompiler code implement this?"*
  - *"What are the internal C++ class names?"*
  - *"Can you provide proprietary vendor keys or certificates?"*
- Any prohibited inquiry will be marked `Rejected`.

### 9.2 Resolution Flow
1. Implementer appends a question using the schema in `spec/questions.md`.
2. The specification team investigates the inquiry using hardware testbenches, network captures, or protocol analysis.
3. The resolution is synthesized into clean, normative language and committed as a version bump to the relevant `spec/*.md` document.
4. `spec/questions.md` is updated with a reference to the resolving commit.

---

## 10. Complete Specification Suite Directory

Implementers should use this index to navigate the specification documents:

| Document | Primary Role in Toolchain | Key Topics |
|---|---|---|
| [`00-system-overview.md`](00-system-overview.md) | Architectural Context | Hardware specs, extraction methodology, daemon inventory, listening ports, boot order. |
| [`10-storage.md`](10-storage.md) | Storage & Directory Maps | Flash layout, `/remote/Store` hierarchy, permissions, staging paths. |
| [`20-upload-manifest.md`](20-upload-manifest.md) | Compiler: Manifest Engine | 32-bit rotating checksum algorithm, staged filename syntax, `.info` format. |
| [`21-containers.md`](21-containers.md) | Compiler: Container Engine | `TMC2` container framing, `ENC DATA` AES-128 ECB + HMAC-SHA256 envelope. |
| [`30-project-model.md`](30-project-model.md) | Compiler: Topology Engine | `mcs_info.dat`, `mcs.dat`, and `trf_dev_db.json` schemas. |
| [`31-automation-tables.md`](31-automation-tables.md) | Compiler: Logic Tables | `Link.dat`, `Event.dat`, `Alarm.dat`, and `FB/*.fb` binary schemas. |
| [`32-download-database.md`](32-download-database.md) | Compiler: Database Engine | `DnData.dn` 56-byte header, 26-table directory, inner room/device/macro records. |
| [`33-master-project-database.md`](33-master-project-database.md) | Compiler: Backup Archive | `CCS/edit.dat.gz` database header, table directory, and EOF sentinel. |
| [`40-modules.md`](40-modules.md) | Compiler: Module Engine | Two-Way Tcl runtime, `::varIf*` globals, driver lifecycle, IP/serial I/O. |
| [`50-discovery.md`](50-discovery.md) | Deployer: Discovery | UDP 61510 `PACKET_FIND` (`0xC350`) / `PACKET_FIND_ACK` (`0xC353`) framing. |
| [`51-upload-protocol.md`](51-upload-protocol.md) | Deployer: Staging & Reload | FTP upload lifecycle, TCP 51520 querying, UDP 61520 reload trigger (`0x2030`). |
| [`52-control-protocol.md`](52-control-protocol.md) | Diagnostic & Runtime Client | TCP 51510 `0x02` framing, opcodes: `0x0FE1` status, `0x089A` variables, `0x07DA` macros. |
| [`53-client-protocols.md`](53-client-protocols.md) | Diagnostic & Mobile Client | Mobile app sync, TCP 51520 `0x2070` pre-registration, TCP 51510 `0x1389` login. |
| [`90-examples/minimal-program.md`](90-examples/minimal-program.md) | Golden Verification Reference | Worked minimal program derivation: complete byte streams and test vectors. |
| [`README.md`](README.md) | Handbook & Standards | Clean-room boundaries, extraction methodology, RFC 2119 keywords, Definition of Done. |
| [`questions.md`](questions.md) | Q&A Communication Channel | Implementer inquiry log and feedback loop. |
