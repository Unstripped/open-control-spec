---
title: "Upload Manifests and Staged File Names"
section: "20-upload-manifest"
version: "0.1.0"
status: review
circumvention: false
---

# Upload Manifests and Staged File Names

## 1. Scope

A programmer delivers a project to the controller by placing files in the staging directory `/remote/Store/updateroot/` (see `10-storage` §3.1 for the transfer path). Every staged file carries a structured name that says who it is for and identifies its content. Manifests (`.info` files) list which staged files make up the current release for one recipient.

This section specifies:

- the staged file name grammar (§2),
- the 32-bit content checksum used in names and manifests (§3),
- the manifest record format (§4) and the position record (§5),
- how the controller uses names and manifests (§6), with errors and edge cases (§7),
- test vectors built from our own inputs (§8).

Confidence tags follow Section 00 §4.5 (`[CONFIRMED]`, `[STATIC]`, `[INFERRED]`).

## 2. Staged file name grammar

### 2.1 Layout

All characters are ASCII. Hex digits MUST be uppercase.

```
name     = prefix "_" address ( ".info" / "_" kind token ( "_" original / ".info" / "_position.info" ) )
prefix   = ALPHA 4HEXDIG                 ; 5 characters
address  = ALPHA 16HEXDIG                ; 17 characters
kind     = "C" / "V" / "S" / "T"
token    = 8HEXDIG
original = 1*VCHAR                       ; the file's own name, e.g. core_model.json
```

| Offset | Size | Field | Type | Allowed values | Description |
|---|---|---|---|---|---|
| 0 | 1 | `prefix_class` | ASCII letter | `M`, `T`, `U`, `O`, `A` (observed) | What the file is. See §2.2. |
| 1 | 4 | `prefix_id` | ASCII hex | `0000`–`FFFF` | Model id for `M`; content group for the other classes. |
| 5 | 1 | separator | ASCII | `_` | The controller ignores a file whose byte 5 is not `_` `[STATIC]`. |
| 6 | 1 | `address_class` | ASCII letter | `A`, `M`, `O`, `Z` | Recipient addressing mode. See §2.3. |
| 7 | 16 | `address_id` | ASCII hex | | Recipient identifier (8 bytes, hex). |
| 23 | 1 | separator | ASCII | `_` or `.` | `.` only in the legacy bare manifest form `<prefix>_<address>.info`. Any other byte makes the controller ignore the file `[STATIC]`. |
| 24 | 1 | `kind` | ASCII letter | `C`, `V`, `S`, `T` | Meaning of `token`. See §2.4. |
| 25 | 8 | `token` | ASCII hex, zero-padded | | 32-bit value. |
| 33 | … | suffix | ASCII | `_<original>`, `.info`, `_position.info` | Content file, manifest, or position record. |

The fixed part is 33 characters. Generators MUST emit exactly these widths; the controller reads fields by position, not by splitting on `_` `[STATIC]`.

### 2.2 Prefix classes

| `prefix_class` | Meaning | Observed `prefix_id` | Confidence |
|---|---|---|---|
| `M` | Manifest owner: the device model the manifest (or file) is addressed to. `prefix_id` is the model id from the model table (`MRX-15=8008`, `Android Tablet=E206`, `IPhoneV2=E207`, `IPadV2=E208`, …). | `8008`, `0000`, `E206`, `E207`, `E208` | `[CONFIRMED]` for the listing behaviour (§6.2) |
| `T` | Client UI content set. | `0002` (phone set), `0003` (tablet set) | `[INFERRED]` grouping |
| `U` | User images (room, custom, favorite images). | `0003` | `[INFERRED]` |
| `O` | Optional fonts. | `0003` | `[INFERRED]` |
| `A` | Content common to all clients (system fonts). | `0000` | `[INFERRED]` |

The controller enumerates content classes in the order `O`, `U`, `A`, `T` when a client asks for the extended file list `[STATIC]`. Content files (`T`, `U`, `O`, `A`) are always addressed `A0000000000000000` in observed uploads; they reach a client only by being named in that client's manifest.

### 2.3 Address classes

| `address_class` | `address_id` | Meaning | Confidence |
|---|---|---|---|
| `A` | 16 × `0` | Every device of the prefix's model. | `[CONFIRMED]` |
| `M` | 12-hex MAC followed by `0000` | One device by MAC. Example: MAC `00:1F:B8:00:11:22` → `M001FB80011220000`. Used for the controller's own project database set. | `[CONFIRMED]` (naming) / `[STATIC]` (controller match) |
| `O` | 8-byte client id whose first 4 bytes are zero, e.g. `0000000000000007` | One client by numeric id. | `[CONFIRMED]` |
| `Z` | 8-byte client id | Same as `O`/`M` but used when the client talks over the Zigbee transport. | `[STATIC]` |

When a client identifies itself with an 8-byte id, the controller forms the client address as `O` if the first four id bytes are zero, else `M`, followed by the 8 id bytes in transmission order as 16 hex digits `[CONFIRMED]` for `O`.

### 2.4 Token kinds

| `kind` | `token` meaning | Used on | Confidence |
|---|---|---|---|
| `C` | Content checksum (§3) of the staged file's bytes. | Content files and per-client `model.json`. | `[CONFIRMED]` |
| `V` | Version: high 16 bits = year, low 16 bits = build number. `V07E617D5` = 2022 / 6101. | Controller program binary. | `[STATIC]` |
| `S` | Serial number, 8 decimal digits. | None observed. | `[STATIC]` |
| `T` | Creation time, Unix seconds (UTC). | Manifests and position records. All manifests of one upload for one recipient group share one value. | `[CONFIRMED]` (`T632B7C9D` = 2022-09-21 21:05:33 UTC) |

## 3. Content checksum

### 3.1 Definition

For a file of bytes `b[0] … b[n-1]`:

```
term(i)  = ( b[i] << (i mod 8) ) | ( b[i] >> (8 - (i mod 8)) )
checksum = ( Σ_{i=0}^{n-1} term(i) ) mod 2^32
```

- `b[i]` is an unsigned byte (0–255).
- Each term is computed in at least 16-bit precision and is **not** truncated to 8 bits. For `i mod 8 = 0` the term is `b[i]` (the right shift by 8 yields 0).
- The sum wraps modulo 2^32.
- `i` is the absolute byte offset in the file. (The controller reads in chunks of 8096 bytes and restarts the index per chunk; 8096 is a multiple of 8, so the result is identical.)
- An empty file has checksum 0.

`[CONFIRMED]`: reproduces the `C` token of 21 of 24 vendor-staged content files, the column-6 value of every installed controller file checked (7 files, up to 13 MB), and the value the controller itself reports for files we staged (§6.2). The three exceptions are in §9.

### 3.2 Rendering

- In a staged name: 8 uppercase hex digits, zero-padded (`C00018B39`).
- In a manifest column: uppercase hex **without** leading zeros (`18B39`); zero renders as `0`.

## 4. Manifest (`.info`) format

### 4.1 File

| Property | Value | Confidence |
|---|---|---|
| Name | `<prefix>_<address>_T<time>.info` (current form). The bare form `<prefix>_<address>.info` is looked up first by the controller and treated as optional (§6.2). | `[CONFIRMED]` |
| Encoding | ASCII | `[CONFIRMED]` |
| Line terminator | CR LF on every record, including the last | `[CONFIRMED]` |
| Header | none | `[CONFIRMED]` |
| Record count | one per staged file; observed 1–12 | `[CONFIRMED]` |
| Maximum size the controller reads | 2048 bytes (for the column-7 lookup), 8192 bytes per manifest in the legacy list filter | `[STATIC]` |

Generators SHOULD keep a manifest under 2048 bytes.

### 4.2 Record

Nine fields separated by a single TAB (0x09). Numbers carry no sign, padding, or prefix.

| Order | Field | Type | Encoding | Allowed values | Description | Confidence |
|---|---|---|---|---|---|---|
| 1 | `is_archive` | uint | ASCII decimal | `0`, `1` | `1` if the file is an archive the recipient unpacks (`.tar`, `.tar.gz`, single-file `.gz`), else `0`. | `[CONFIRMED]` (all 46 vendor records) |
| 2 | `archive_ordinal` | uint | ASCII decimal | `0` for non-archives; `1…` for archives | Running count of archive records. Client manifests restart at 1 in each manifest. The controller's own manifests in one upload continue the count across manifests. | `[CONFIRMED]` values / `[INFERRED]` meaning |
| 3 | `staged_name` | string | ASCII | §2 | Name of the staged file this record describes. | `[CONFIRMED]` |
| 4 | `size` | uint | ASCII decimal | | Byte length of the staged file. | `[CONFIRMED]` |
| 5 | `version` | uint32 | ASCII hex, no leading zeros | | Equals the token for `V` files; `0` otherwise. | `[CONFIRMED]` |
| 6 | `checksum` | uint32 | ASCII hex, no leading zeros | | §3 checksum of the staged bytes. Equals the token for `C` files. | `[CONFIRMED]` |
| 7 | `payload_checksum` | uint32 | ASCII hex, no leading zeros | | For a single-file gzip: §3 checksum of the decompressed payload. For `.tar` archives in the controller set: non-zero, derivation unknown (§9). Otherwise `0`. | `[CONFIRMED]` for gzip, unknown for tar |
| 8 | Unknown / Reserved | string | ASCII | always `N/A` | Unknown. Generators MUST write `N/A`. | `[CONFIRMED]` value |
| 9 | Unknown / Reserved | uint | ASCII decimal | always `0` | Unknown. Generators MUST write `0`. | `[CONFIRMED]` value |

Every vendor record re-renders byte-identically from these nine parsed fields `[CONFIRMED]`.

### 4.3 Which manifests an upload contains

Observed upload, per recipient group:

| Manifest | Lists |
|---|---|
| `M<client model>_A0000000000000000_T<t>.info` | The shared content set for that client model (UI model, button table, images, theme, fonts, module lists). |
| `M<client model>_O<client id>_T<t>.info` | That client's own `model.json` (staged as `M<client model>_O<client id>_C<sum>_model.json`). |
| `M<client model>_O<client id>_T<t>_position.info` | §5. |
| `M8008_A0000000000000000_T<t>.info` | Controller-wide files (program binary, Tcl package, sub-processor image, `rms.ini`, `prov_model.ini`). |
| `M8008_M<mac>0000_T<t>.info` | This controller's project database files (`DnData.dn`, `FB.tar`, `mcs.dat`, `mcs_info.dat`, `TwoWay.tar`). |
| `M0000_A0000000000000000_T<t>.info` | `edit.dat.gz` (programmer project backup). |

The same content file MAY be listed by several client manifests and MUST be staged once.

## 5. Position record (`_position.info`)

| Property | Value |
|---|---|
| Name | `M<client model>_O<client id>_T<t>_position.info`, same `<t>` as that client's manifest |
| Content | one line: `<room name>` TAB `<device label>` CR LF |
| Example | `Living Room\tAndroid Tablet\r\n` |

`[CONFIRMED]` (3 vendor records; the controller lists ours to the client, §6.2).

## 6. Controller behaviour

### 6.1 The checksum is an identity label, not a gate

The controller does not compare a staged file's `C` token or manifest column 6 against the file content before serving or installing it `[STATIC]`. It exposes the checksum to clients on request (§6.2), and it uses tokens to tell versions of the same file apart (§6.3). Generators MUST still compute tokens per §3: clients compare them with the controller-reported checksum.

### 6.2 Client file listing (TCP 51520)

The full exchange belongs to `51-upload-protocol`. Facts needed to stage files:

- A client asks with its model id and 8-byte client id. The controller answers with every staged file whose name **starts with** `M<model>_A0000000000000000` or with the client's address (§2.3), each followed by a 32-bit value `[CONFIRMED]`. Content files are not listed directly; the client reads the listed manifests to find them.
- A client id of all zeros matches every `O`-addressed file of that model `[CONFIRMED]`.
- The 32-bit value per listed file is meaningful only for `V` files (the token) and for files whose name contains `MB101` or `C100` (models THZ-100 and TRC-780), where it is a checksum; for every other file it is left over from earlier work and MUST be ignored by clients `[CONFIRMED]` (stale value observed) / `[STATIC]` (THZ-100/TRC-780 case).
- A separate checksum query returns the §3 checksum of each named staged file, or `0xFFFFFFFF` for a file it cannot read `[CONFIRMED]`.
- Before listing, the controller tries the bare manifest names `<prefix>_<address>.info`. If either is missing it lists files unfiltered `[CONFIRMED]`. With both present it drops every listed name that is neither a `.info` file nor mentioned in one of the two bare manifests `[STATIC]`. Generators SHOULD NOT emit bare-form manifests.

### 6.3 Superseded files

When two staged files share the 23-character `prefix_address` and the same original name, the controller deletes the one whose 8-hex token is numerically smaller `[STATIC]`. For this comparison it strips a trailing `.tar.gz` or `.gz` from the original name and compares by prefix, so `fonts.tar.gz` also collides with a later `fonts.tar` `[STATIC]`. Consequence: generators MUST give a replacement file a different token from the file it replaces, and SHOULD delete the old file themselves rather than rely on this.

### 6.4 Controller program and database install

For files addressed to the controller (`M8008`, `M0000`):

- The controller installs a file only if its full staged name is not already listed in `/remote/Store/Prg/version.info`; it concatenates installed manifests (`cat A M > /remote/Store/Prg/version.info`) after a successful run `[CONFIRMED]`. A changed token or missing entry is what triggers reinstallation.
- It requires an `M8008_M<own MAC>…*.info` and an `M8008_A…*.info` in `/remote/Store/updateroot/` `[CONFIRMED]`; if absent or unreadable, the install aborts without updating.
- The install action is chosen by the original name substring (`DnData.dn`, `FB.tar`, `TwoWay.tar`, `rms.ini`, `BS.exe`, `tclpkg.tar.gz`, `subproc.*`, `edit.dat.gz`). `mcs.dat`, `mcs_info.dat`, and link tables are bypassed by the staging installer `[CONFIRMED]`. Detailed semantics belong to `51-upload-protocol`.
## 7. Errors and edge cases

| Input | Controller behaviour | Confidence |
|---|---|---|
| Byte 5 not `_`, or byte 23 not `_`/`.` | File ignored in client listings. | `[STATIC]` |
| Name shorter than 23 characters | Payload-checksum lookup refuses it. | `[STATIC]` |
| `C` token does not match content | Served unchanged; the checksum query reports the true checksum, so a client can detect the mismatch. Observed on 3 vendor files. | `[CONFIRMED]` |
| Manifest names a file that is not staged | Not checked by the controller. Checksum query returns `0xFFFFFFFF` for it. | `[STATIC]` |
| Manifest longer than 2048 bytes | Payload-checksum lookup reads only the first 2048 bytes. | `[STATIC]` |
| Hex digits in lowercase | Prefix matching is case-sensitive; such files are not listed. | `[INFERRED]` |
| Directory or symlink in the staging directory | Skipped by listings (regular files only). | `[STATIC]` |
| An interrupted remote-management download marker is present at service start | The controller deletes everything in the staging directory. | `[STATIC]` |

## 8. Test vectors

All inputs are synthetic test vectors generated for clean-room conformance validation.

### 8.1 Checksum

Input bytes `55 52 43 2D 4D 52 58 21 80 FF` (`URC-MRX!` then `0x80 0xFF`):

| i | byte | i mod 8 | term |
|---|---|---|---|
| 0 | 0x55 | 0 | 0x0055 |
| 1 | 0x52 | 1 | 0x00A4 |
| 2 | 0x43 | 2 | 0x010D |
| 3 | 0x2D | 3 | 0x0169 |
| 4 | 0x4D | 4 | 0x04D4 |
| 5 | 0x52 | 5 | 0x0A4A |
| 6 | 0x58 | 6 | 0x1616 |
| 7 | 0x21 | 7 | 0x1090 |
| 8 | 0x80 | 0 | 0x0080 |
| 9 | 0xFF | 1 | 0x01FF |

Sum = **0x00003BB2**.

| Input | Length | Checksum |
|---|---|---|
| empty | 0 | `00000000` |
| 100 × `0x00` | 100 | `00000000` |
| 8 × `0xFF` | 8 | `0000FEF8` |
| 1 MiB × `0xFF` (wraps: 131072 × 0xFEF8) | 1048576 | `FDF00000` |
| `hello from test suite\n` | 22 | `0000B9AB` |
| `{"id":7,"ui":"lab"}\n` | 20 | `00008D32` |
| `{"rooms":[{"name":"Lab","devices":[]}]}\n` | 40 | `00018B39` |

### 8.2 Manifest and listing (`[CONFIRMED]` in emulation)

Inputs: `model.json` = `{"id":7,"ui":"lab"}\n`; time 1800000000 (`6B49D200`); recipient model `E206`, client id `0000000000000007`.

Staged files:

```
ME206_O0000000000000007_C00008D32_model.json
ME206_O0000000000000007_T6B49D200.info
ME206_O0000000000000007_T6B49D200_position.info
```

`ME206_O0000000000000007_T6B49D200.info`, 68 bytes:

```
30 09 30 09 4D 45 32 30 36 5F 4F 30 30 30 30 30
30 30 30 30 30 30 30 30 30 30 37 5F 43 30 30 30
30 38 44 33 32 5F 6D 6F 64 65 6C 2E 6A 73 6F 6E
09 32 30 09 30 09 38 44 33 32 09 30 09 4E 2F 41
09 30 0D 0A
```

`ME206_O0000000000000007_T6B49D200_position.info`: `Lab\tTest Tablet\r\n`.

Observed: a client listing for model `E206`, id `0000000000000007` returns these three names (plus the model-wide manifest when staged), and the checksum query for `ME206_O0000000000000007_C00008D32_model.json` returns `0x00008D32`.

## 9. Open items

- **User-image archive tokens.** For the three vendor-staged `U0003_*.tar` files, the `C` token does not equal the §3 checksum of the staged bytes, and the controller itself reports the §3 value. The programmer computes these tokens from something else, possibly its source image set. This does not block staging: generators following §3 produce tokens the controller agrees with. Verify with the PC programmer or a capture of a fresh upload with known images.
- **Column 7 for `.tar` archives.** `FB.tar` and `TwoWay.tar` carry non-zero values whose derivation is unknown (the installed copies changed after install, so they cannot be recomputed). Generators SHOULD write `0` for tar archives until resolved; the controller reads column 7 only for THZ-100/TRC-780 files, and parses it as decimal there.
- **Columns 8 and 9.** Meaning unknown; always `N/A` and `0`.
- **`archive_ordinal` semantics.** Not consumed by the controller code examined; client use unknown.
- **Controller-set install path (`[STATIC]` only).** §6.4 is from static analysis; confirm on the live MRX in network testing. Also trace what moves `M8008`/`M0000` manifests into `/remote/Store/data/`.
- **Prefix classes and groups.** Meaning of `T0002`/`T0003`/`U0003`/`O0003`/`A0000` beyond the observed client split is inferred.
