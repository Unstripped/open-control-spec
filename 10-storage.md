---
title: "Persistent Storage Architecture & Lifecycle"
section: "10-storage"
version: "0.1.0"
status: draft
type: Architecture
confidence: confirmed
circumvention: false
open_items:
  - "Determine whether BS.exe permits bypassing client UI asset staging when programming a headless controller deployment."
  - "Validate whether updateroot/ staging files can be purged automatically after client touchscreens synchronize, or if they must persist indefinitely."
---

# Persistent Storage Architecture & Lifecycle

## 1. Overview & Partition Architecture

### 1.1 Architectural Scope & Interoperability Objective

URC Total Control network controllers (including the MRX-15, MRX-30, MRX-10, and MRX-8) maintain all user configuration, automation logic, compiled project databases, driver runtime packages, and user interface definitions on non-volatile persistent flash memory `[CONFIRMED]`.

The objective of this specification is to define the physical partition layout, filesystem hierarchy, staging pathways, directory contracts, and read/write lifecycles of persistent storage across all subdirectories of `/remote/Store`. This document establishes the vendor-independent technical manual required for an open-source programming client to:
1. Stage project packages and assets to the controller over local network protocols without modifying core operating system binaries.
2. Generate valid `.info` manifest records, configuration databases, and driver archives required for controller activation.
3. Understand the atomic transition from network staging (`updateroot/`) to active runtime execution (`data/`, `Prg/`, `CCS/`).
4. Avoid contaminating or corrupting persistent controller runtime services, security certificates, or factory recovery fallback trees.

### 1.2 Physical Storage & eMMC Partition Layout

The physical controller hardware incorporates an onboard 4 GB eMMC 5.0 flash memory device mapped to block device node `/dev/mmcblk3` `[CONFIRMED]`. The eMMC storage is partitioned into four primary MBR partitions designed to isolate immutable operating system code from mutable automation programming and factory recovery baselines:

```
+---------------------------------------------------------------------------------------------------+
|                                  PHYSICAL eMMC 4GB (/dev/mmcblk3)                                 |
+---------------------------------------------------------------------------------------------------+
| Start LBA | End LBA  | Sectors   | Capacity  | Partition Node | Filesystem | Mount / Role         |
|-----------|----------|-----------|-----------|----------------|------------|----------------------|
| 0         | 0        | 1         | 512 B     | /dev/mmcblk3   | MBR        | Master Boot Record   |
| 1         | 8191     | 8,191     | 4.0 MiB   | /dev/mmcblk3   | Raw        | U-Boot SPL & Image   |
| 8192      | 2105343  | 2,097,152 | 1.0 GiB   | /dev/mmcblk3p1 | ext4       | / (rootfs1, ro)      |
| 2105344   | 4202495  | 2,097,152 | 1.0 GiB   | /dev/mmcblk3p2 | ext4       | Standby rootfs2 (ro) |
| 4202496   | 8396799  | 4,194,304 | 2.0 GiB   | /dev/mmcblk3p3 | ext4       | /media (datafs, rw)  |
| 8396800   | 14688255 | 6,291,456 | 3.0 GiB   | /dev/mmcblk3p4 | ext4       | /backup (golden, ro) |
+---------------------------------------------------------------------------------------------------+
```

#### Partition Roles & Mount Contracts

1. **Partition 1 (`/dev/mmcblk3p1` - `rootfs1`):**
   - 1.0 GiB ext4 filesystem mounted read-only (`ro`) at root (`/`) during standard operation `[CONFIRMED]`.
   - Contains the Linux 4.1.42-fslc kernel modules, GNU C Library 2.25 shared libraries, system utilities, and SysV init scripts (`/etc/init.d/`, `/etc/rcS.d/`, `/etc/rc5.d/`).
   - An open-source programmer MUST NOT attempt to write directly to Partition 1.

2. **Partition 2 (`/dev/mmcblk3p2` - `rootfs2`):**
   - 1.0 GiB ext4 filesystem retained as an unmounted standby recovery rootfs for dual-bank A/B firmware updates `[CONFIRMED]`.

3. **Partition 3 (`/dev/mmcblk3p3` - `datafs`):**
   - 2.0 GiB ext4 filesystem mounted read-write (`rw`) at `/media` by `/etc/init.d/mrx-init-pre.sh` `[CONFIRMED]`.
   - Accommodates all persistent controller state, custom programming, user assets, and staging files.
   - If unformatted or corrupted upon boot, the kernel initialization script formats the partition as ext4 using `mkfs.ext4` `[CONFIRMED]`.
   - Host to all bind mounts that expose persistent state into standard Linux system paths.

4. **Partition 4 (`/dev/mmcblk3p4` - `backup`):**
   - 3.0 GiB ext4 filesystem mounted read-only (`ro`) at `/backup` `[CONFIRMED]`.
   - Houses the golden factory default image (`/backup/backup/`).
   - When a factory reset is triggered (either via kernel command line `factory_reset=1` or boot environment flag `backup_lock`), the initialization script wipes `/media/*` and executes `rsync -a --delete /backup/backup/ /media` `[CONFIRMED]`.

### 1.3 Filesystem Hierarchy & Mount Binding Model

To provide a consistent directory structure for application daemons while keeping the root filesystem read-only, the system utilizes Linux directory bind mounts established during early boot (`rcS.d/S08mrx-init-pre.sh` and `rcS.d/S59mrx-init-post.sh`):

```
+-----------------------------------------------------------------------------------+
|                        PHYSICAL DISK MOUNT: /media                                |
|                        (/dev/mmcblk3p3 - 2 GiB ext4)                              |
+-----------------------------------------------------------------------------------+
       |                    |                    |                     |
       | bind-mount         | bind-mount         | bind-mount          | bind-mount
       v                    v                    v                     v
+--------------+     +--------------+     +------------------+  +------------------+
| /media/remote|     |  /media/lib  |     | /media/interfaces|  | /media/resolv.conf|
+--------------+     +--------------+     +------------------+  +------------------+
       |                    |                    |                     |
       v                    v                    v                     v
+--------------+     +--------------+     +------------------+  +------------------+
|   /remote    |     |   /var/lib   |     |/etc/network/     |  | /etc/resolv.conf |
|              |     |  (SSH keys)  |     | interfaces       |  |                  |
+--------------+     +--------------+     +------------------+  +------------------+
       |
       +--> /remote/Store (Persistent Application Root)
                 |
                 +--> /remote/App (Symlink -> /remote/Store/App)
```

The specific bind-mount mappings established at boot are defined in the following table:

| Physical Source Path | Virtual Mount Target | Permissions | Lifecycle & Purpose |
|---|---|---|---|
| `/dev/mmcblk3p3` | `/media` | `rw,relatime` | Base persistent storage root on eMMC partition 3 `[CONFIRMED]`. |
| `/dev/mmcblk3p4` | `/backup` | `ro,relatime` | Factory golden recovery repository `[CONFIRMED]`. |
| `/media/remote` | `/remote` | `rw,bind` | Application runtime root directory `[CONFIRMED]`. |
| `/media/lib` | `/var/lib` | `rw,bind` | System state persistence; preserves Dropbear SSH host keys across reboots `[CONFIRMED]`. |
| `/media/interfaces` | `/etc/network/interfaces` | `rw,bind` | Persistent network configuration (static IP, netmask, gateway) `[CONFIRMED]`. |
| `/media/resolv.conf` | `/etc/resolv.conf` | `rw,bind` | Persistent DNS nameserver configuration `[CONFIRMED]`. |
| `/remote/Store/updateroot` | `/remote/Store/updateroot_anon/updateroot` | `ro,bind` | Read-only bind mount for anonymous FTP staging jail isolation `[CONFIRMED]`. |

### 1.4 Persistent Application Root (`/remote/Store`) Structure

The directory `/remote/Store` (physically `/media/remote/Store`) serves as the root container for all automation application software, configuration databases, execution frameworks, and staging trees `[CONFIRMED]`. A top-level symbolic link `/remote/App -> /remote/Store/App` provides standard Unix execution paths for supervisory scripts.

`/remote/Store` contains eight primary functional subdirectories:

1. **`updateroot/` (Staging Area):** Staging target directory where programming tools upload new project archives, UI assets, driver bundles, and `.info` manifests via FTP (TCP port 21).
2. **`data/` (Active Project Database & Hardware State):** Active runtime configuration files (`DnData.dn`, `mcs.dat`, `mcs_info.dat`), feedback tables (`FB/`), Two-Way driver packages (`TwoWay/`), transient state (`TclTemp/`), and logging (`UrcDebugging/`).
3. **`Prg/` (Program Execution & Module Framework):** Engine binary copy (`BS.exe`), Tcl package libraries (`tclpkg/`), optional shared libraries (`lib/`), and administrative control tokens (`ssh_enable.info`).
4. **`CCS/` (Complete Control Suite Database):** Compressed Accelerator 3 master project database (`edit.dat.gz`).
5. **`Common/` (Cross-Module Shared State):** Shared persistent runtime state exchanged between independent Two-Way device modules across restarts (e.g. Sonos zone MAC mapping tables).
6. **`Favorite/` (UI Graphics Cache):** Cached graphical icons for favorite channels, services, and room shortcuts.
7. **`App/` (System Daemons & Platform Configuration):** System daemon executables (`BS.exe`, `deviceguard`, `devmngd.exe`, `rui_svrd.exe`, `svcd.exe`), FPGA hardware bitstream (`MRX_30.rbf`), supervision policies (`service.ini`), network server configurations (`vsftpd.conf.*`, `mosquitto.conf`), and X.509 security certificates.
8. **`updateroot_anon/` (FTP Chroot Jail):** Anonymous FTP landing directory configured with an internal mount point `updateroot` bound to `/remote/Store/updateroot`.

---

## 2. Master Storage Inventory

The table below catalogs every file pattern, directory family, and configuration artifact residing within `/remote/Store`. It defines the producing writer, consuming readers, write timing, replacement behavior during a project upload, programmer production requirements, and the corresponding roadmap specification unit:

- **Programmer Requirement:**
  - **Mandatory:** An open-source programmer MUST generate this file for the controller to execute the program.
  - **Optional:** Produced when corresponding features (custom fonts, audio zones, Two-Way IP modules) are present in the project.
  - **Forbidden:** System binaries, security keys, or platform configs that an open-source programmer MUST NOT overwrite.
  - **None:** Dynamically generated by controller daemons or drivers at runtime.

| Path / File Pattern | Producing Writer | Consuming Readers | Write Timing / Lifecycle | Program Upload Replacement | Programmer Requirement | Specification Section |
|---|---|---|---|---|---|---|
| `HDA_REQ_JSON.json` | `BS.exe` / Audio IPC | `BS.exe`, audio daemons | Runtime operation | Runtime (dynamic state cache) | None | N/A (Internal IPC) `[CONFIRMED]` |
| `updateroot/*.info` | Programmer / Dealer Tool | `BS.exe` (`CheckRLS`) | Staging phase (FTP TCP 21) | Yes (replaced every upload) | **Mandatory** | `20-upload-manifest` `[CONFIRMED]` |
| `updateroot/*_position.info` | Programmer / Dealer Tool | `rui_svrd.exe`, client devices | Staging phase (FTP TCP 21) | Yes (replaced every upload) | **Mandatory** | `20-upload-manifest` `[CONFIRMED]` |
| `updateroot/direct_download` | `rmsmcsman.exe` / Tool | `rmsmcsman.exe`, `BS.exe` | During active file staging | Runtime (transient marker) | None | `51-upload-protocol` `[CONFIRMED]` |
| `updateroot/*_TCLImages.tar.gz` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (replaced every upload) | **Mandatory** | `53-client-protocols` `[CONFIRMED]` |
| `updateroot/*_Theme.tar.gz` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (replaced every upload) | **Mandatory** | `53-client-protocols` `[CONFIRMED]` |
| `updateroot/*_fonts.tar.gz` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (replaced when fonts change)| **Optional** | `53-client-protocols` `[CONFIRMED]` |
| `updateroot/*_sys_fonts.tar.gz` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (system font baseline) | **Mandatory** | `53-client-protocols` `[CONFIRMED]` |
| `updateroot/*_room_images.tar` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (replaced every upload) | **Optional** | `53-client-protocols` `[CONFIRMED]` |
| `updateroot/*_custom_images.tar` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (replaced every upload) | **Optional** | `53-client-protocols` `[CONFIRMED]` |
| `updateroot/*_fav_images.tar` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (replaced every upload) | **Optional** | `53-client-protocols` `[CONFIRMED]` |
| `updateroot/*_model.json` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (replaced every upload) | **Mandatory** | `53-client-protocols` `[CONFIRMED]` |
| `updateroot/*_core_model.json` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (replaced every upload) | **Mandatory** | `53-client-protocols` `[CONFIRMED]` |
| `updateroot/*_ui_button_table.json` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (replaced every upload) | **Mandatory** | `53-client-protocols` `[CONFIRMED]` |
| `updateroot/*_modules.lst` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (replaced every upload) | **Mandatory** | `40-modules` `[CONFIRMED]` |
| `updateroot/*gui.vfd.tar` | Programmer / Dealer Tool | Client touchscreens, `rui_svrd.exe` | Staging phase (FTP TCP 21) | Yes (per Two-Way device) | **Optional** | `40-modules` `[CONFIRMED]` |
| `data/*.info` | Programmer / Dealer Tool | `BS.exe` | Activation / Archive | Yes (manifests copied from staging) | **Optional** | `20-upload-manifest` `[CONFIRMED]` |
| `data/DnData.dn` | `BS.exe` (`CheckRLS`) | `BS.exe` (Automation core) | Activation phase | Yes (moved from `updateroot/`) | **Mandatory** (staged) | `32-download-database` `[CONFIRMED]` |
| `data/mcs.dat` | `BS.exe` (`CheckRLS`) | `BS.exe`, `rui_svrd.exe`, `devmngd.exe` | Activation phase | Yes (moved from `updateroot/`) | **Mandatory** (staged) | `30-project-model` `[CONFIRMED]` |
| `data/mcs_info.dat` | `BS.exe` (`CheckRLS`) | `BS.exe`, `rui_svrd.exe` | Activation phase | Yes (moved from `updateroot/`) | **Mandatory** (staged) | `30-project-model` `[CONFIRMED]` |
| `data/Link.dat` | `BS.exe` / Programmer | `BS.exe` | Activation / Runtime | Yes (overwritten on activation)| **Mandatory** | `31-automation-tables` `[CONFIRMED]` |
| `data/Event.dat` | `BS.exe` / Programmer | `BS.exe` | Activation / Runtime | Yes (overwritten on activation)| **Mandatory** | `31-automation-tables` `[CONFIRMED]` |
| `data/Alarm.dat` | `BS.exe` / Programmer | `BS.exe` | Activation / Runtime | Yes (overwritten on activation)| **Mandatory** | `31-automation-tables` `[CONFIRMED]` |
| `data/FB/*.fb` | `BS.exe` (`CheckRLS`) | `BS.exe`, HDA audio handlers | Activation phase | Yes (extracted from `FB.tar`) | **Optional** (staged) | `31-automation-tables` `[CONFIRMED]` |
| `data/TwoWay/*.vfd/**` | `BS.exe` (`CheckRLS`) | `BS.exe` (Tcl driver host) | Activation phase | Yes (extracted from `TwoWay.tar`)| **Optional** (staged) | `40-modules` `[CONFIRMED]` |
| `data/prov_model.ini` | Factory baseline / `BS.exe` | `provisioningd`, `BS.exe` | Boot / Firmware update | No (system baseline catalog) | **Optional** | `30-project-model` `[CONFIRMED]` |
| `data/rms.ini` | Programmer / `rmsmcsman.exe`| `rmsmcsman.exe`, `BS.exe` | Activation / Runtime | Optional (RMS enabled flag) | **Optional** | `30-project-model` `[CONFIRMED]` |
| `data/trf_dev_db.json` | `BS.exe` / RF Subsystem | `BS.exe` | Runtime operation | Runtime (dynamic RF state) | None | `30-project-model` `[CONFIRMED]` |
| `data/multi_push_info.dat` | `BS.exe` (push registration)| `BS.exe` | Runtime operation | No (preserved across uploads) | None | `53-client-protocols` `[CONFIRMED]` |
| `data/ccsid_check.dat` | `BS.exe` (`CheckRLS`) | `BS.exe` | Activation phase | Yes (generated verification) | None | `33-master-project-database` `[CONFIRMED]` |
| `data/deviceguard_log.txt` | `deviceguard` | Diagnostics | Runtime operation | Runtime (log stream) | None | N/A (Diagnostics) `[CONFIRMED]` |
| `data/TclTemp/**` | Two-Way Tcl Drivers | Two-Way Tcl Drivers | Runtime operation | Runtime (transient scratchpad) | None | `40-modules` `[CONFIRMED]` |
| `data/UrcDebugging/**` | Two-Way Tcl Drivers | `logmanager.exe` / Diagnostics| Runtime operation | Runtime (driver debug logs) | None | `40-modules` `[CONFIRMED]` |
| `CCS/edit.dat.gz` | `BS.exe` (`CheckRLS`) | `BS.exe`, Dealer TCP 51511 | Activation phase | Yes (moved from `updateroot/`) | **Mandatory** (staged) | `33-master-project-database` `[CONFIRMED]` |
| `Prg/BS.exe` | Firmware installer | Kernel / `S60RCMain.sh` | Firmware upgrade | No (system binary copy) | **Forbidden** | N/A (System Binary) `[CONFIRMED]` |
| `Prg/tclpkg/**` | `BS.exe` (`CheckRLS`) | `BS.exe` (Tcl interpreter) | Activation / Firmware init | No (framework runtime library)| **Optional** (staged) | `40-modules` `[CONFIRMED]` |
| `Prg/tclpkg.tar.gz` | `BS.exe` (`CheckRLS`) | `BS.exe` | Activation phase | No (archive cache) | **Optional** (staged) | `40-modules` `[CONFIRMED]` |
| `Prg/lib/**` | Firmware installer | Dynamic linker (`ld-linux`) | Firmware upgrade | No (shared library overrides) | **Forbidden** | N/A (System Infrastructure) `[CONFIRMED]` |
| `Prg/ssh_enable.info` | Admin / Developer | `svcd.exe` / `dropbear` | Diagnostic configuration | No (persistent diagnostic flag) | None | N/A (Platform Admin) `[CONFIRMED]` |
| `Prg/version.info` | `BS.exe` (`CheckRLS`) | `BS.exe` | Post-activation | Runtime (concatenated installed manifests) | None | `51-upload-protocol` `[CONFIRMED]` |
| `Common/**` | Two-Way Tcl Drivers | Two-Way Tcl Drivers | Runtime operation | Runtime (cross-session state) | None | `40-modules` `[CONFIRMED]` |
| `Favorite/**` | UI Touchscreens / Programmer| `rui_svrd.exe`, touchscreens | Staging / Runtime | Yes (UI graphics cache) | **Optional** | `53-client-protocols` `[CONFIRMED]` |
| `App/**` (Daemons) | Factory baseline / Firmware | SysV init, `svcd.exe` | Boot / Firmware upgrade | No (operating system daemons) | **Forbidden** | N/A (System Infrastructure) `[CONFIRMED]` |
| `App/MRX_30.rbf` | Factory baseline / Firmware | `S50RCInit.sh` -> sysfs | Boot (FPGA co-processor) | No (hardware bitstream) | **Forbidden** | N/A (Hardware Bitstream) `[CONFIRMED]` |
| `App/service.ini` | Factory baseline / Firmware | `svcd.exe` | Boot / Supervisor init | No (supervisory policy) | **Forbidden** | N/A (System Infrastructure) `[CONFIRMED]` |
| `App/dwnsuccess.ini` | `BS.exe` | Dealer tool / `rmsmcsman.exe` | Post-activation | Runtime (timestamp of reload) | None | `51-upload-protocol` `[CONFIRMED]` |
| `App/client_download.info` | `BS.exe` | Diagnostics / Auditing | Runtime client pull | Runtime (download log) | None | `53-client-protocols` `[CONFIRMED]` |
| `App/preference.db` | `rui_svrd.exe` | Touchscreens / Mobile clients | Runtime operation | Runtime (UI room visibility) | None | `53-client-protocols` `[CONFIRMED]` |
| `App/mrx-rootfs.ext4` | Factory baseline | OS recovery subsystem | Factory recovery | No (loopback recovery image) | **Forbidden** | N/A (System Infrastructure) `[CONFIRMED]` |
| `App/ccc`, `App/ccpv` | Factory provisioning | `deviceguard` | Commissioning & pairing | No (X.509 device credentials) | **Forbidden** | N/A (Platform Security) `[CONFIRMED]` |
| `App/vsftpd.pem` | Factory provisioning | `vsftpd` | Boot (SSL/TLS staging) | No (FTP TLS certificate) | **Forbidden** | N/A (Platform Security) `[CONFIRMED]` |

---

## 3. Directory Contracts & File Format Specifications

### 3.1 Staging Root (`updateroot/` and `updateroot_anon/`)

#### 3.1.1 Staging Directory Contract

The directory `/remote/Store/updateroot` is the designated landing target for all project staging transfers. Staging files are delivered via standard FTP (TCP port 21). When connecting anonymously, `vsftpd` places the connection into `/remote/Store/updateroot_anon`, where an internal directory `/remote/Store/updateroot_anon/updateroot` is bind-mounted directly to `/remote/Store/updateroot` `[CONFIRMED]`.

#### 3.1.2 Staged File Names, Manifests, and Position Records

Staged file names, the content checksum, the `.info` manifest record format, and `*_position.info` records are specified in `20-upload-manifest`.

---

### 3.2 Active Project Database & Runtime State (`data/`)

The directory `/remote/Store/data` contains all active runtime state consumed by the automation engine (`BS.exe`), device manager (`devmngd.exe`), and user interface daemon (`rui_svrd.exe`) `[CONFIRMED]`.

#### 3.2.1 Core Database Files

1. **`DnData.dn` (Controller Download Database):**
   - Binary database compiled by the programming tool containing device tables, room definitions, infrared pulse sequence tables, RS-232 command strings, and macro execution trees `[CONFIRMED]`.
   - Begins with a 32-byte binary header followed by controller model string identifier (e.g. `MRX-15\x00`).
   - Sourced from staged file `M{MODEL}_M{MAC}_C{CHKSUM}_DnData.dn`.

2. **`mcs.dat` (Device & Audio Zone Model):**
   - JSON-formatted structured database cataloging connected devices, IR database codes, IP control parameters, and audio zones `[CONFIRMED]`.
   - Read by `BS.exe` to instantiate zone models and audio routing matrices.
   - Sourced from staged file `M{MODEL}_M{MAC}_C{CHKSUM}_mcs.dat`.

3. **`mcs_info.dat` (Base Station & Client Topology):**
   - JSON-formatted system metadata defining the primary master controller (`bstations`), connected client interfaces (`clients`), device list (`devices`), programming software version (`editor_ver`), and download timestamp `[CONFIRMED]`.
   - Sourced from staged file `M{MODEL}_M{MAC}_C{CHKSUM}_mcs_info.dat`.

4. **`Link.dat`, `Event.dat`, `Alarm.dat`:**
   - Text configuration tables defining variable link rules, event trigger actions, and scheduled real-time clock alarms `[CONFIRMED]`.

5. **`ccsid_check.dat`:**
   - 4-byte binary verification token written by `BS.exe` upon successful project activation. Used on startup to verify synchronization between `data/` and `CCS/edit.dat.gz` `[CONFIRMED]`.

6. **`prov_model.ini`:**
   - INI-style hardware lookup catalog mapping commercial model names to 16-bit hexadecimal hardware identifiers `[CONFIRMED]`:
     ```ini
     MRX-15=0x8008
     MRX-30=0x8007
     Android Tablet=0xE206
     IPhoneV2=0xE207
     IPadV2=0xE208
     ```

7. **`multi_push_info.dat`:**
   - Binary state cache storing Google/Firebase Cloud Messaging (FCM) registration tokens uploaded by mobile handsets `[CONFIRMED]`.
   - Preserved across program uploads so client notifications function without re-pairing.

#### 3.2.2 Audio Zone Feedback Directory (`data/FB/`)

The directory `/remote/Store/data/FB` contains individual zone channel mapping descriptors unpacked from staged file `M{MODEL}_M{MAC}_C{CHKSUM}_FB.tar`:
- Files are named by three-digit zone index: `{ZONE_ID}.fb` (e.g. `010.fb`, `018.fb`, `026.fb`).
- File contents contain plain text channel routing definitions:
  - `ZONE=1LR`: Standard stereo channel pair `[CONFIRMED]`.
  - `ZONE=1L Mono`: Left channel summed mono `[CONFIRMED]`.
  - `ZONE=1R Mono`: Right channel summed mono `[CONFIRMED]`.

#### 3.2.3 Two-Way Driver Package Directory (`data/TwoWay/`)

The directory `/remote/Store/data/TwoWay` houses runtime driver packages unpacked from staged file `M{MODEL}_M{MAC}_C{CHKSUM}_TwoWay.tar`:
- Each driver resides in a subdirectory named `{MODULE_NAME}.vfd/` (e.g. `SonosV2.vfd/`).
- Package contents:
  - `info.ini`: Driver metadata, authoring version, supported models, and network protocols.
  - `{MODULE_NAME}.tcl`: Main Tcl driver scripts executed within `BS.exe`'s embedded interpreter.
  - Driver helper scripts (`systemSDK.tcl`, `URC_API.tcl`, parameters, playlist managers).

#### 3.2.4 Dynamic Runtime State (`TclTemp/` and `UrcDebugging/`)

- **`data/TclTemp/{MODULE}/{OBJ_ID}/`:** Dynamic scratchpad created at runtime by active Tcl driver threads (e.g. discovery state files, loopback command descriptors) `[CONFIRMED]`.
- **`data/UrcDebugging/LOG/{MODULE}/{OBJ_ID}/`:** Rolling runtime debug log files (`{MODULE}.log`, `{MODULE}.log.1` ... `{MODULE}.log.10`) maintained with 512 KB size rotation limits `[CONFIRMED]`.
- **`data/UrcDebugging/LOG/Level/{MODULE}/{OBJ_ID}/`:** Dynamic log level modifiers (`fileLevel.mod`, `syslogLevel.mod`, `shellLevel.mod`) controlling verbosity `[CONFIRMED]`.

---

### 3.3 Program Executables & Support Libraries (`Prg/`)

The directory `/remote/Store/Prg` contains application binaries and runtime support packages:

1. **`Prg/BS.exe`:**
   - Standalone copy of the core automation engine binary, byte-identical to `/remote/Store/App/BS.exe` `[CONFIRMED]`.
   - Updated during firmware maintenance packages staged as `_BS.exe` or `subproc.tar.gz`.

2. **`Prg/tclpkg/` and `Prg/tclpkg.tar.gz`:**
   - Tcl support libraries required by Two-Way device modules (tcllib 1.16, sqlite3 3.13.0, syslog 1.1) `[CONFIRMED]`.
   - Accompanied by a version metadata stamp `Prg/tclpkg/version` (e.g. `2.0.2022.0427.1`).
   - Extracted from `M8008_A0000000000000000_CAC2525AD_tclpkg.tar.gz` during framework updates.

3. **`Prg/ssh_enable.info`:**
   - Zero-byte marker file created by administrators to prevent watchdog teardown of Dropbear SSH console access `[CONFIRMED]`.
   - Persists across project reloads and cold boots.

---

### 3.4 Project Storage (`CCS/`)

The directory `/remote/Store/CCS` contains the master project database file:
- **`CCS/edit.dat.gz`:**
  - Gzip-compressed binary file containing the complete Accelerator 3 project database `[CONFIRMED]`.
  - Uncompressed stream begins with binary header `}\x07\x00\x00...` and version token `CCP_BIN_VER_STR3.02.0628.4` `[CONFIRMED]`.
  - Contains complete system topology: room hierarchy, device instances, macro definitions, graphical layout coordinates, client assignments, and original project filename (e.g. `Sample_Project.tc3`).
  - Downloaded by programming software over TCP port 51511 when performing project retrieval/backup operations.

---

### 3.5 Cross-Module Shared State (`Common/`)

The directory `/remote/Store/Common` provides persistent, non-volatile state storage shared across independent Two-Way device modules and surviving controller reboots `[CONFIRMED]`:
- Path hierarchy: `/remote/Store/Common/{MODULE_NAME}/{OBJ_ID}/` (e.g. `/remote/Store/Common/SonosV2/11/`).
- Stored records:
  - `hardZones.txt`: Text mapping of zone identifiers to device MAC addresses (e.g. `1 001122334455`).
  - `hardZonesMac.txt`: Inverted MAC-to-zone index lookup table.
  - `instructions.txt`: Module operational state and synchronization counter.

---

### 3.6 User Interface Favorites (`Favorite/`)

The directory `/remote/Store/Favorite` acts as a filesystem cache for user-selected television channels, streaming services, and favorite macro shortcut icons:
- Populated when client touchscreens or mobile apps save custom channel favorites `[CONFIRMED]`.
- Served dynamically by `rui_svrd.exe` to connected touchscreen clients.

---

### 3.7 Master Application & System Assets (`App/`)

The directory `/remote/Store/App` houses master system executables, co-processor bitstreams, and platform configurations:
- **System Executables:** `BS.exe` (automation core), `deviceguard` (security and licensing), `devmngd.exe` (physical I/O), `rui_svrd.exe` (remote UI server), `rui_svrd_launcher.exe`, `svcd.exe` (supervisor), `early_init`, `vsftpd`, `mosquitto`.
- **FPGA Bitstream:** `MRX_30.rbf` (Altera Cyclone IV configuration bitstream loaded into hardware via sysfs during boot).
- **Supervisory Configuration:** `service.ini` (defines auto-start daemons and port 22222 socket activation).
- **Service Configurations:** `mosquitto.conf`, `vsftpd.conf.*` (FTP configuration templates).
- **Runtime Databases & System Images:**
  - `preference.db`: SQLite database storing user-selected room visibility state for touchscreens `[CONFIRMED]`.
  - `mrx-rootfs.ext4`: Loopback ext4 recovery image (64 MiB) `[CONFIRMED]`.
- **State Records & Audit Logs:**
  - `dwnsuccess.ini`: Timestamp file updated upon each successful project activation and engine reload `[CONFIRMED]`.
  - `client_download.info`: Audit log tracking client device downloads of UI staging bundles `[CONFIRMED]`.
- **Security Assets:**
  - `ccc`: X.509 device pairing certificate `[CONFIRMED]`.
  - `ccpv`: Encrypted private key corresponding to `ccc` `[CONFIRMED]`.
  - `vsftpd.pem`: TLS certificate and key for FTPS connections `[CONFIRMED]`.
An open-source programming client MUST NOT alter files in `/remote/Store/App/` during standard program uploads `[CONFIRMED]`.

---

## 4. Staging vs Active Runtime Synchronization Lifecycle

### 4.1 Lifecycle State Machine

The transition of program configurations from network upload to active execution proceeds through five deterministic lifecycle phases:

```
+-----------------------------------------------------------------------------------+
| 1. STAGING PHASE (FTP TCP 21)                                                     |
|    - Programmer connects to vsftpd (anonymous or local user)                      |
|    - Staging assets placed in /remote/Store/updateroot/                           |
|    - Runtime daemons continue normal execution from data/, Prg/, and CCS/         |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| 2. ACTIVATION TRIGGER (TCP 51510)                                                 |
|    - Programmer sends binary activation trigger frame to BS.exe                   |
|    - BS.exe enters staging verification state (CheckRLS module)                   |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| 3. MANIFEST SELECTION (see 20-upload-manifest §6.4)                               |
|    - BS.exe requires M8008_M<MAC> and M8008_A .info manifests to be readable      |
|    - Skips files whose staged name is already recorded as installed              |
|    - No checksum or size verification of staged files [STATIC]                    |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| 4. ATOMIC EXTRACTION & MIGRATION                                                  |
|    - M0000_..._edit.dat.gz moved to /remote/Store/CCS/edit.dat.gz                 |
|    - M8008_..._DnData.dn moved to /remote/Store/data/DnData.dn                    |
|    - M8008_..._mcs.dat and mcs_info.dat moved to /remote/Store/data/              |
|    - M8008_..._FB.tar extracted into /remote/Store/data/FB/                       |
|    - M8008_..._TwoWay.tar extracted into /remote/Store/data/TwoWay/               |
|    - BS.exe generates /remote/Store/data/ccsid_check.dat verification stamp        |
|    - System sync() issued to flush writes to eMMC flash                           |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| 5. ENGINE RELOAD & CLIENT SYNCHRONIZATION                                         |
|    - BS.exe reloads project models and restarts Tcl driver processes              |
|    - rui_svrd.exe notifies touchscreens and mobile clients of new project hash    |
|    - Clients download updated assets from /remote/Store/updateroot/ via FTP/HTTP  |
+-----------------------------------------------------------------------------------+
```

### 4.2 Staging Phase Mechanics

1. **Transport & Authentication:** Staging uploads occur over FTP (TCP port 21). The controller supports both anonymous logins and local authenticated logins `[CONFIRMED]`.
2. **Directory Resolution:** Anonymous logins land in `/remote/Store/updateroot_anon`, where subfolder `updateroot/` is bind-mounted to `/remote/Store/updateroot`. Standard logins land in `/remote/Store/` and navigate directly into `updateroot/` `[CONFIRMED]`.
3. **Non-Disruptive Staging:** Active automation services (macro execution, IR/RS-232 command dispatch, touchscreen UI rendering) continue uninterrupted during file upload.

### 4.3 Manifest Selection

Manifest handling and the content checksum are specified in `20-upload-manifest` (§6.4 for this install path). The controller does not verify checksums or sizes of staged files before install `[STATIC]`. Failure reporting over the network is specified in `51-upload-protocol`.

### 4.4 Atomic Migration & Unpack Rules

For each selected staged file, `BS.exe` installs it by original name:
- **Plain Files (Archive Flag = `0`):** Copied byte-for-byte from `/remote/Store/updateroot/{STAGED_NAME}` to the destination chosen by the original name (e.g. `/remote/Store/data/DnData.dn`) `[STATIC]`.
- **Archives (Archive Flag = `1`):** Extracted using standard tar commands:
  - `FB.tar`: Extracted into `/remote/Store/data/FB/` `[CONFIRMED]`.
  - `TwoWay.tar`: Extracted into `/remote/Store/data/TwoWay/` `[CONFIRMED]`.
  - `tclpkg.tar.gz`: Extracted into `/remote/Store/Prg/` `[CONFIRMED]`.
- **Verification Stamp:** `BS.exe` calculates the 32-bit project identifier from `edit.dat.gz` and writes it as a 4-byte little-endian token to `/remote/Store/data/ccsid_check.dat` `[CONFIRMED]`.
- **Filesystem Commit:** `BS.exe` executes a filesystem synchronization (`sync`) to ensure all filesystem metadata and directory changes are committed to eMMC blocks `[CONFIRMED]`.

---

## 5. Programmer File Production Matrix

To successfully program an MRX controller, an open-source programming client MUST produce a defined set of files and stage them to `/remote/Store/updateroot/`. The table below outlines every artifact the programmer must generate, categorized by necessity and linking to the responsible roadmap specification unit:

| Artifact Category | File Identifier / Pattern | Mandatory / Optional | Target Destination on Device | Specification Section |
|---|---|---|---|---|
| **Master Project DB** | `M0000_A0000000000000000_C{HASH}_edit.dat.gz` | **Mandatory** | `/remote/Store/CCS/edit.dat.gz` | `33-master-project-database` `[CONFIRMED]` |
| **Controller Manifest**| `M8008_M{MAC}0000_T{TIME}.info` | **Mandatory** | `/remote/Store/data/` | `20-upload-manifest` `[CONFIRMED]` |
| **Global Manifest** | `M0000_A0000000000000000_T{TIME}.info` | **Mandatory** | `/remote/Store/data/` | `20-upload-manifest` `[CONFIRMED]` |
| **Download Database** | `M8008_M{MAC}0000_C{HASH}_DnData.dn` | **Mandatory** | `/remote/Store/data/DnData.dn` | `32-download-database` `[CONFIRMED]` |
| **Device Model JSON** | `M8008_M{MAC}0000_C{HASH}_mcs.dat` | **Mandatory** | `/remote/Store/data/mcs.dat` | `30-project-model` `[CONFIRMED]` |
| **Topology JSON** | `M8008_M{MAC}0000_C{HASH}_mcs_info.dat` | **Mandatory** | `/remote/Store/data/mcs_info.dat` | `30-project-model` `[CONFIRMED]` |
| **Feedback Archive** | `M8008_M{MAC}0000_C{HASH}_FB.tar` | **Optional** (Audio) | Extracted to `/remote/Store/data/FB/` | `31-automation-tables` `[CONFIRMED]` |
| **Two-Way Drivers** | `M8008_M{MAC}0000_C{HASH}_TwoWay.tar` | **Optional** (Modules) | Extracted to `/remote/Store/data/TwoWay/` | `40-modules` `[CONFIRMED]` |
| **Client Manifests** | `ME{MODEL}_O{ID}_T{TIME}.info` | **Mandatory** (Clients) | `/remote/Store/updateroot/` | `20-upload-manifest` `[CONFIRMED]` |
| **Client Positions** | `ME{MODEL}_O{ID}_T{TIME}_position.info` | **Mandatory** (Clients) | `/remote/Store/updateroot/` | `20-upload-manifest` `[CONFIRMED]` |
| **UI Graphics Bundles**| `T{TYPE}_A0000000000000000_C{HASH}_TCLImages.tar.gz` | **Mandatory** (UI) | `/remote/Store/updateroot/` | `53-client-protocols` `[CONFIRMED]` |
| **UI Themes** | `T{TYPE}_A0000000000000000_C{HASH}_Theme.tar.gz` | **Mandatory** (UI) | `/remote/Store/updateroot/` | `53-client-protocols` `[CONFIRMED]` |
| **UI Core Models** | `T{TYPE}_A0000000000000000_C{HASH}_core_model.json` | **Mandatory** (UI) | `/remote/Store/updateroot/` | `53-client-protocols` `[CONFIRMED]` |
| **UI Button Tables** | `T{TYPE}_A0000000000000000_C{HASH}_ui_button_table.json` | **Mandatory** (UI) | `/remote/Store/updateroot/` | `53-client-protocols` `[CONFIRMED]` |
| **System Fonts** | `A0000_A0000000000000000_C{HASH}_sys_fonts.tar.gz` | **Mandatory** (UI) | `/remote/Store/updateroot/` | `53-client-protocols` `[CONFIRMED]` |

---

## 6. Open Items

The following architectural and empirical verification tasks remain open:
- [x] Checksum algorithm and `.info` record format: resolved in `20-upload-manifest`; remaining manifest gaps are listed there.
- [ ] Determine whether `BS.exe` permits bypassing client UI asset staging when programming a headless controller deployment.
- [ ] Validate whether `updateroot/` staging files can be purged automatically after client touchscreens synchronize, or if they must persist indefinitely.
