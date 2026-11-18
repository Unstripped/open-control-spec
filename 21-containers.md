---
title: "Containers, Module Packaging, and Encryption Formats"
section: 21
status: draft
---

# 21. Containers, Module Packaging, and Encryption Formats

> **LEGAL NOTICE & NOMINATIVE FAIR USE DISCLAIMER:**  
> This document is part of an independent, clean-room technical specification authored solely to achieve device and network interoperability. This project is not affiliated with, sponsored by, or endorsed by Universal Remote Control, Inc. (URC). All product names, trademarks, and registered trademarks (including "URC", "Total Control", "MRX-8", "MRX-10", "MRX-15", "MRX-30", and "Accelerator") are the property of their respective owners, and are used here exclusively for nominative identification and technical compatibility purposes under 15 U.S.C. § 1125 and controlling fair use doctrines.

## 1. Overview

URC Total Control MRX controllers consume Two-Way device drivers, scripts, and runtime modules packaged in two distinct container formats:
1. **TMC2 Archive (`.tmc2` / `URC_COLLECT_SEC`):** A multi-file binary container format used by programmer suites to distribute driver packages containing component databases (`.cd2`), Two-Way virtual front device packages (`.vfd`), and UI manifests (`info.dat`, `desktop.ini`).
2. **`ENC DATA` Authenticated Stream Container:** A binary envelope wrapping executable Tcl scripts (`.tcl`) deployed under `/remote/Store/data/TwoWay/<Module>.vfd/`.

This section defines the file layouts, field tables, encryption and key derivation pipelines, and plaintext acceptance behavior required for an independent programmer to author and deploy valid module packages.

### 1.1 Mandatory External Key Provisioning Model (Zero-Key Contract)

Key handling model for conforming tools:
1. **Zero Key Distribution:** This specification details container framing, header layouts, block sizes, and cipher modes (AES-128 ECB / CBC) in the abstract. ZERO proprietary cryptographic keys, initialization vectors, seed materials, or certificates are distributed within this specification or any compliant reference tooling.
2. **User-Supplied Runtime Injection:** Compliant compiler or unpacker implementations MUST NOT hardcode, embed, or automatically fetch proprietary keys. Any tool that encrypts content authored by its operator MUST require the operator to supply the key at runtime via:
   - Environment variables (e.g., `OPENCONTROL_CONTENT_KEY`, `OPENCONTROL_PROJECT_KEY`), or
   - A secure local key file (`--key-file /path/to/key.bin`).
3. **Graceful Degradation:** If the required external key is not provided at runtime, tools MUST terminate cleanly with an informative error stating that key material was not supplied by the operator, and MUST NOT fall back to any embedded secret.

---

## 2. TMC2 Container Format (`URC_COLLECT_SEC`)

### 2.1 File Architecture

A `.tmc2` container comprises three contiguous sections:
1. **Header Block (`0x00 - 0x1A`):** 27-byte header declaring the format magic, version, flags, filename field width, and total entry count.
2. **Record Table (`0x1B - 0x461A`):** Exactly 128 fixed 140-byte slot descriptors (17,920 bytes total).
3. **Payload Data (`0x461B - EOF`):** Contiguous member file payloads, aligned to 16-byte boundaries.

```
+-------------------------------------------------------------------+
| Header Block (27 bytes, 0x00 - 0x1A)                              |
+-------------------------------------------------------------------+
| Slot Descriptors [0 .. 127] (17,920 bytes, 0x1B - 0x461A)         |
|   - Slots 0 .. N-1: Populated file descriptors                    |
|   - Slots N .. 127: Empty slot descriptors                        |
+-------------------------------------------------------------------+
| Member Payloads (Offset 0x461B to EOF)                            |
|   - Each payload aligned to 16-byte boundary                      |
+-------------------------------------------------------------------+
```

### 2.2 Header Block Table

| Offset | Size | Type | Encoding | Field | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `0x00` | 16 | string | ASCII | `magic` | Must be ASCII `"URC_COLLECT_SEC\0"`. |
| `0x10` | 2 | uint16 | Little-Endian | `version` | Container version. MUST be `0x0200` (`512`). |
| `0x12` | 1 | uint8 | Unsigned | `flags` | Sub-version flags. MUST be `0x00`. |
| `0x13` | 4 | uint32 | Little-Endian | `filename_len`| Length of the filename buffer per slot. MUST be `128` (`0x00000080`). |
| `0x17` | 4 | uint32 | Little-Endian | `total_entries`| Number of active populated entries $N$ ($1 \le N \le 128$). |

### 2.3 Slot Descriptor Table

Each descriptor occupies exactly 140 bytes (`filename_len + 12`). The table always contains 128 entries regardless of the number of populated files.

#### Populated Entry ($0 \le i < N$)

| Offset | Size | Type | Encoding | Field | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `+0x00` | 128 | string | ASCII | `filename` | Null-terminated relative file path. Unused bytes MUST be zero-padded. |
| `+0x80` | 4 | uint32 | Little-Endian | `alloc_size` | Allocated payload block size. MUST be 16-byte aligned: `(size + 15) & ~15`. |
| `+0x84` | 4 | uint32 | Little-Endian | `offset` | Absolute byte offset in the archive where the member payload starts. |
| `+0x88` | 4 | uint32 | Little-Endian | `size` | Exact logical length of the member payload in bytes. |

#### Empty Entry ($N \le i < 128$)

| Offset | Size | Type | Encoding | Field | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `+0x00` | 128 | bytes | Binary | `filename` | MUST be all zero bytes (`0x00`). |
| `+0x80` | 4 | uint32 | Little-Endian | `alloc_size` | MUST be `0x00000000`. |
| `+0x84` | 4 | uint32 | Little-Endian | `offset` | MUST be `0x00000000`. |
| `+0x88` | 4 | uint32 | Little-Endian | `size` | MUST be `0x00000005` (empty-slot marker). |

### 2.4 Payload Alignment

- Every member payload MUST start at a 16-byte aligned offset.
- Because the record table ends at offset `17,947` (`0x461B`), the first payload begins at offset `17,947`.
- Trailing padding bytes between logical `size` and `alloc_size` MUST be zero-filled (`0x00`).

### 2.5 Member Payload Encryption (`.TCM*` Suffixes)

Member files inside `.tmc2` archives carry format-specific extension suffixes denoting payload encryption:

| Member Pattern | Suffix | Content Type | Cipher |
| :--- | :--- | :--- | :--- |
| `*.cd2.TCM0` | `.TCM0` | Component Database (device commands, zones, variables) | 128-bit block cipher, ECB mode `[INFERRED]` |
| `*.vfd.TCM1` | `.TCM1` | Virtual Front Device (Tcl scripts, UI definitions) | 128-bit block cipher, ECB mode `[INFERRED]` |
| `*.ini.TCM-1` | `.TCM-1` | Metadata mirror / fallback descriptor | 128-bit block cipher, ECB mode `[INFERRED]` |
| `info.dat` | None | Package manifest | High-entropy stream `[INFERRED]` |

#### 2.5.1 Cipher and Block Geometry
- **Cipher Mode:** 16-byte block cipher in ECB mode (AES-128) `[INFERRED]`.

#### 2.5.2 Scope
- **Controller Boundary:** The controller does not read `.tmc2` or `.TCM*` files; they are consumed only by the dealer PC programming software `[INFERRED]`.
- **PC Programmer Boundary:** `.tmc2` is a driver distribution container used by the dealer PC programming software.
- **Deployment on Controller:** When a module is uploaded to the controller, the PC programmer packages it into `M8008_..._TwoWay.tar`. The controller extracts this tar archive into `/remote/Store/data/TwoWay/<Module>.vfd/`, where `.tcl` scripts are encrypted in `ENC DATA` format using the project security key (see Section 3).
- **Interoperability Requirement:** An independent open-source programmer targeting the MRX controller does not need to produce `.tmc2` files; it compiles modules directly into `TwoWay.tar` with `ENC DATA` encapsulation.

---

## 3. `ENC DATA` Authenticated Encryption Format

### 3.1 Envelope Structure

Encrypted Two-Way Tcl files use the `ENC DATA` binary format. All multi-byte integers in the `ENC DATA` header are serialized in **Big-Endian** byte order.

```
+-------------------------------------------------------------------+
| Magic: "ENC DATA" (8 bytes, 0x00 - 0x07)                          |
+-----------------------------------+-------------------------------+
| Header Length = 60 (0x08 - 0x0B)  | Version = 0x1000 (0x0C - 0x0F)|
+-----------------------------------+-------------------------------+
| Reserved Zero Padding (36 bytes, 0x10 - 0x33)                     |
+-----------------------------------+-------------------------------+
| Plaintext Size (0x34 - 0x37)      | Ciphertext Indicator (38 - 3B)|
+-----------------------------------+-------------------------------+
| HMAC-SHA256 Authentication Tag (32 bytes, 0x3C - 0x5B)            |
+-------------------------------------------------------------------+
| AES-128 Ciphertext (Offset 0x5C to EOF)                           |
+-------------------------------------------------------------------+
```

### 3.2 Header Layout

| Offset | Size | Type | Encoding | Field | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `0x00` | 8 | string | ASCII | `magic` | Must be ASCII `"ENC DATA"`. |
| `0x08` | 4 | uint32 | Big-Endian | `header_len` | Size of the header block. MUST be `60` (`0x0000003C`). |
| `0x0C` | 4 | uint32 | Big-Endian | `version` | Format version. MUST be `0x00001000` (`4096`). |
| `0x10` | 36 | bytes | Binary | `reserved` | Reserved. MUST be all zero bytes (`0x00`). |
| `0x34` | 4 | uint32 | Big-Endian | `plain_len` | Uncompressed logical plaintext length in bytes. |
| `0x38` | 4 | uint32 | Big-Endian | `enc_len` | Ciphertext length indicator. Value is `ciphertext_len + 32`. |
| `0x3C` | 32 | bytes | Binary | `hmac` | HMAC-SHA256 authentication tag computed over padded plaintext. |
| `0x5C` | $M \times 16$ | bytes | Binary | `ciphertext` | AES-128 ECB encrypted blocks. |

---

## 4. Key Derivation and Cryptographic Operations

A symmetric encryption key is required.

### 4.1 Key Hierarchy

Two keys are involved in Two-Way module execution:
1. **Controller Content Key:** A fixed 16-byte key held by the controller. The controller uses it to read the Table 0 protected block of `DnData.dn` ([Download Database](32-download-database.md) §4.1). This specification does not provide it or describe how to obtain it.
2. **Project Security Key:** A 16-byte key chosen by the programmer for each project and delivered to the controller in the Table 0 protected block. The controller uses it to decrypt Two-Way module `.tcl` scripts at boot (§5). A programmer encrypts its own module scripts with this key using §4.2–§4.4.

### 4.2 Key Derivation Pipeline

Both AES encryption and HMAC integrity verification share a single 16-byte derived key $K_{derived}$ generated from the 16-byte raw key $K_{raw}$:

1. **Buffer Preparation:** Staging buffer $B$ of 32 bytes is allocated:
   $$B[0 \dots 15] = K_{raw}, \quad B[16 \dots 31] = 0x00$$
2. **Iterative Hashing:** A SHA-256 hash context is initialized and updated with $B$ exactly 117 times (`0x75` in hex):
   $$\text{digest} = \text{SHA-256}^{117}(B)$$
   More specifically, the SHA-256 compression function is fed the same 32-byte block $B$ in 117 consecutive update calls before finalizing.
3. **Key Selection:** The first 16 bytes of the resulting SHA-256 digest are taken as the derived key:
   $$K_{derived} = \text{digest}[0 \dots 15]$$

### 4.3 Integrity Authentication (HMAC-SHA256)

1. The plaintext $P$ is padded to a 16-byte block boundary using standard PKCS#7 padding:
   $$\text{pad\_len} = 16 - (\text{len}(P) \pmod{16})$$
   $$P_{padded} = P \parallel \text{bytes}([\text{pad\_len}] \times \text{pad\_len})$$
2. The authentication tag is computed via HMAC-SHA256 using $K_{derived}$ over $P_{padded}$:
   $$\text{HMAC} = \text{HMAC-SHA256}(K_{derived}, P_{padded})$$

### 4.4 Cipher Operation (AES-128 ECB)

Each 16-byte block of $P_{padded}$ is encrypted using AES in Electronic Codebook (ECB) mode with key $K_{derived}$:
$$C_i = \text{AES-128-ECB-Encrypt}(K_{derived}, P_{padded, i})$$

---

## 5. Controller Execution & Plaintext Acceptance Rules

The controller startup and execution logic enforces the following rules for module scripts:

1. **Staged Location:** Module directories reside at `/remote/Store/data/TwoWay/<Module>.vfd/`.
2. **Decryption Cycle:** At controller boot, the master daemon scans `/remote/Store/data/TwoWay/` for `.vfd` directories, creates a matching `/tmp/<Module>.vfd/` directory on volatile storage, and processes every `.tcl` file.
3. **Plaintext Rejection in TwoWay Directory:**
   - The decryption routine inspects the first 8 bytes of every `/remote/Store/data/TwoWay/*/*.tcl` file.
   - If the header does not match ASCII `"ENC DATA"`, the file is rejected.
   - The target file in `/tmp/` is left empty or uncreated.
   - The runtime interpreter evaluates `/tmp/<Module>.vfd/*.tcl` using standard `Tcl_EvalFile`.
   - **Conclusion:** Files installed into `/remote/Store/data/TwoWay/` **MUST be encrypted** using the `ENC DATA` format. Plaintext `.tcl` files placed into `/remote/Store/data/TwoWay/` are not executed by the controller.
4. **Project Database Formats:**
   - `mcs.dat`, `mcs_info.dat`, and `trf_dev_db.json` are standard plaintext UTF-8 JSON files.
   - `Link.dat`, `Event.dat`, and `Alarm.dat` are standard unencrypted binary tables.
   - `CCS/edit.dat.gz` is a standard gzip archive.

---

## 6. Test Vectors

### 6.1 TMC2 Container Test Vector

A minimal valid `.tmc2` container holding a single 13-byte member file (`info.dat` containing ASCII `"TITLE=Test\r\n"`):

- **Header (`0x00 - 0x1A`):**
  ```
  00000000: 55 52 43 5f 43 4f 4c 4c 45 43 54 5f 53 45 43 00  URC_COLLECT_SEC.
  00000010: 00 02 00 80 00 00 00 01 00 00 00                 ...........
  ```
- **Slot 0 (`0x1B - 0xA6`):**
  - Filename: `"info.dat\0"` (128 bytes, zero-padded)
  - `alloc_size`: `0x00000010` (16 bytes)
  - `offset`: `0x0000461B` (17,947)
  - `size`: `0x0000000D` (13 bytes)
- **Slots 1 to 127 (`0xA7 - 0x461A`):**
  - Empty slots: 128 zero bytes, followed by `alloc_size = 0`, `offset = 0`, `size = 5`.
- **Payload (`0x461B - 0x462A`):**
  ```
  54 49 54 4c 45 3d 54 65 73 74 0d 0a 00 00 00 00
  ```
- **Total file size:** `17,963` bytes (`0x462B`).

### 6.2 `ENC DATA` Encryption Test Vector

Given:
- Input plaintext $P$ (13 bytes): `"puts \"Hello\"\n"` (`70 75 74 73 20 22 48 65 6c 6c 6f 22 0a`)
- A raw 16-byte key $K_{raw}$ is required.
- Padding (3 bytes of `0x03`): `70 75 74 73 20 22 48 65 6c 6c 6f 22 0a 03 03 03`
- Big-Endian Header fields:
  - `header_len` = `60` (`0x0000003C`)
  - `version` = `4096` (`0x00001000`)
  - `plain_len` = `13` (`0x0000000D`)
  - `enc_len` = `48` (`0x00000030`, representing 16 bytes ciphertext + 32)
- Output size: $60 + 32 + 16 = 108$ bytes.

---

## 7. Open Items

1. **Client UI image formats:** Whether client remote apps parse `.tmc2` archives directly or receive extracted files will be specified in [Client Protocols](53-client-protocols.md).
2. **`.tmc2` member encryption:** Performed by the vendor PC software and out of scope; an independent programmer deploys modules as `TwoWay.tar` with `ENC DATA` (§3).
