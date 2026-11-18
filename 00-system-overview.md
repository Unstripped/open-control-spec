---
title: "System Overview & Process Architecture"
section: "00-system-overview"
version: "0.6.0"
status: review
type: Architecture
confidence: confirmed
open_items:
  - "Empirically decode the binary serialization structure of /tmp/dobj_*.cmd dynamic UI command files."
  - "Map full JSON request/response schema for DeviceGuard REST endpoints on 127.0.0.1:54211."
  - "Verify trigger packet layout for on-demand socket activation daemon svc_exam on TCP port 22222."
  - "Determine whether MQTT header.seq is ever non-zero; every observed publish carried seq 0."
  - "Identify a state-query path for rooms and HDA zones unchanged since boot; MQTT retains only values published since broker start."
  - "Resolve hda/zone roomindex semantics and the mapping between hda/ampstatus zone numbers and hda/zone zone components."
---

# System Overview & Process Architecture

> **LEGAL NOTICE & NOMINATIVE FAIR USE DISCLAIMER:**  
> This document is part of an independent, clean-room technical specification authored solely to achieve device and network interoperability. This project is not affiliated with, sponsored by, or endorsed by Universal Remote Control, Inc. (URC). All product names, trademarks, and registered trademarks (including "URC", "Total Control", "MRX-8", "MRX-10", "MRX-15", "MRX-30", and "Accelerator") are the property of their respective owners, and are used here exclusively for nominative identification and technical compatibility purposes under 15 U.S.C. § 1125 and controlling fair use doctrines.
## 1. System Overview

### 1.1 Architectural Scope & Interoperability Objective

URC Total Control network controllers (including the MRX-15, MRX-30, MRX-10, and MRX-8) serve as the central automation processors within residential and commercial automation deployments. The primary controller executes scheduling and logic engines, maintains device state models, coordinates bidirectional communication with IP peripherals, controls local physical I/O (infrared, RS-232 serial, dry contact relays, 12V voltage sensors), and serves remote user interface (RUI) definitions to native touchscreens, mobile handsets, and handheld remote controls.

The objective of this specification is to define the runtime architecture, initialization sequence, userland process topology, network listening sockets, inter-process communication (IPC) pathways, and external network dependencies of the controller. This document establishes the vendor-independent technical foundation necessary for third-party, open-source programming suites to discover controllers, stage automation projects, activate runtime configurations, and monitor system health over a local network.

### 1.2 Hardware Platform & Operating Environment

The physical controller hardware and operating environment are characterized by the following specifications:

- **Main Application Processor:** NXP / Freescale i.MX6 DualLite application processor (`MCIMX6U5DVM10AC`), incorporating dual ARM Cortex-A9 MPCore execution cores operating at a nominal clock frequency of 792 MHz (configurable up to 996 MHz) with 32 KB instruction and data L1 caches and 512 KB unified L2 cache `[CONFIRMED]`.
- **System Memory:** 1 GB DDR3 SDRAM clocked at 400 MHz (800 MT/s) `[CONFIRMED]`.
- **Persistent Non-Volatile Storage:** 4 GB eMMC 4.41 flash memory module partitioned into system rootfs, data store, and factory recovery banks `[CONFIRMED]`.
- **Hardware Acceleration Co-Processor:** Altera (Intel) Cyclone IV E FPGA attached to the processor on an internal memory-mapped platform bus. The FPGA co-processor handles real-time physical I/O acceleration, including infrared carrier modulation (38 kHz to 56 kHz), IR learning pulse capture, high-speed multi-channel UART FIFOs, sensor edge interrupt arbitration, and relay switching logic `[CONFIRMED]`.
- **Operating System Kernel:** Embedded Linux kernel 4.1.42-fslc compiled for the `armv7-a` architecture with hardware floating-point support (`armhf` / `armv7at2hf-neon-fslc-linux-gnueabi`) `[CONFIRMED]`.
- **C Runtime & Core Toolchain:** GNU C Library (`glibc`) version 2.25 built from the Yocto Project (Pyro 2.3.2 release) `[CONFIRMED]`.
- **System Initialization:** Standard SysV Init (`init [5]`) managing deterministic multi-user runlevel execution `[CONFIRMED]`.

### 1.3 System Execution Topology

The controller runtime executes across six distinct sequential and hierarchical layers:

```
+-----------------------------------------------------------------------------------+
| 1. Bootloader & Kernel (U-Boot -> Linux 4.1.42 -> /sbin/init [5])                 |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| 2. Storage Binding (persistent storage mounted; /remote bound from it;            |
|    tmpfs -> /tmp)                                                                 |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| 3. Hardware & Co-Processor Init (FPGA co-processor and GPIO initialization)       |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| 4. Network & Core Infrastructure (networking, udhcpc, dropbear, dhcpmngd,         |
|    provisioningd UDP 61510, vsftpd TCP 21, mosquitto TCP 1883)                    |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| 5. Process Supervision (svcd.exe /remote/App/service.ini)                         |
|    - Auto-start background daemons (deviceguard, devmngd.exe, mdnsd, etc.)        |
|    - On-demand socket activation triggers (port 22222 -> svc_exam)                |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| 6. Automation Core & UI Servers                                                   |
|    - BS.exe: Automation engine, macro executor, driver host (TCP 51510/51511)     |
|    - rui_svrd.exe: UI server for touchscreens and mobile clients (TCP 51530)      |
+-----------------------------------------------------------------------------------+
```


### 1.4 Protocol Verification & Interoperability Methodology

All behaviors in this specification were established from observable controller behavior: network packet captures, filesystem state observed over the controller's own file-transfer and shell services, controlled requests against its network services, and byte-exact round-trip tests of independently written parsers and generators. Each claim carries a confidence tag ([CONFIRMED], [STATIC], [INFERRED]) as defined in §4 item 5 of the [Specification Handbook](README.md).

---

## 2. Boot Sequence & Service Lifecycle

### 2.1 Boot Sequence Phases

The boot sequence progresses deterministically from hardware power-on to full automation readiness across defined initialization phases:

| Phase | Init Mechanism | Component / Target | Primary Purpose | Dependencies | Execution Mode & Timing |
|---|---|---|---|---|---|
| **0: Kernel & Init** | Bootloader / init | Linux kernel | Boots the kernel and starts SysV init | Hardware power | Synchronous `[CONFIRMED]` |
| **1: Storage** | Early init scripts | Persistent storage | Mounts persistent storage, binds `/remote`, mounts `tmpfs` for `/tmp` | Kernel init | Synchronous `[CONFIRMED]` |
| **2: Network & Diagnostics** | Runlevel 5 scripts | `udhcpc`, `dropbear`, `syslogd` | DHCP lease on `eth0`; SSH on TCP 22; syslog to `/var/log/messages` | Storage | Asynchronous `[CONFIRMED]` |
| **3: Base Daemons & Supervisor** | Runlevel 5 scripts | `dhcpmngd`, `provisioningd`, `svcd.exe /remote/App/service.ini` | Discovery on UDP 61510; supervisor parses `service.ini`, starts immediate services, binds TCP 22222 for on-demand activation; `early_init` replaces the initial FTP server with `vsftpd` on TCP 21 serving `/remote/Store/updateroot` | Network | Asynchronous `[CONFIRMED]` |
| **4: Automation & UI** | Runlevel 5 / `svcd.exe` | `BS.exe`, `rui_svrd_launcher.exe`, `rui_svrd.exe` | `BS.exe` opens TCP 51510/51511 and loads `/remote/Store/Prg`; UI listener on TCP 51530 plus per-client renderer ports | Supervisor | Persistent `[CONFIRMED]` |

### 2.2 Storage Binding & Filesystem Hierarchy

The storage subsystem relies on physical partition separation and Linux bind mounts to decouple the read-only operating system image from mutable user programming data and persistent configuration:

| Block Device / Path | Filesystem | Mount Target | Persistence | Description |
|---|---|---|---|---|
| `/media/remote` | (Bind Mount) | `/remote` | Read-Write (Application Root) | Application runtime root directory. Contains active automation executables, driver packages, and user configuration databases `[CONFIRMED]`. |
| `/remote/Store` | Directory | `/remote/Store` | Read-Write (Data Store) | Persistent storage tree holding binary applications (`App/`), automation logic (`Prg/`), configuration data (`data/`), and staging files (`updateroot/`) `[CONFIRMED]`. |
| `/remote/Store/App` | Symlink | `/remote/App` | Read-Write (Execution Tree) | Convenience symbolic link pointing `/remote/App` directly to `/remote/Store/App` `[CONFIRMED]`. |
| `/remote/Store/updateroot` | Directory | `/remote/Store/updateroot` | Read-Write (Staging Root) | Target directory for anonymous and authenticated FTP project staging uploads from programming software `[CONFIRMED]`. |
| `tmpfs` | `tmpfs` | `/var/volatile` | Volatile (RAM Disk) | Ephemeral RAM disk for high-frequency logs, lock files, and inter-process command buffers. Symlinked via `/tmp -> /var/tmp` `[CONFIRMED]`. |

### 2.3 FPGA Co-Processor

An FPGA co-processor provides IR, serial, and relay timing for the physical interfaces. It is initialized at boot and is not configured by programming software.

### 2.4 Supervision Architecture (`service.ini` Schema & Startup Policies)

The service supervisor daemon (`svcd.exe`) manages secondary userland processes and provides on-demand socket activation. Its configuration file is located at `/remote/App/service.ini` `[CONFIRMED]`.

#### 2.4.1 Configuration File Format

The configuration file is a whitespace-delimited tabular text file. Lines beginning with `#` are treated as comments and MUST be ignored by parsers. Blank lines MUST be ignored `[CONFIRMED]`.

Each valid directive entry contains four whitespace-separated fields:

```
#daemon                             port     max      auto
/remote/App/early_init              0        1        1
/remote/App/deviceguard             0        1        1
/remote/App/svc_exam                22222    1        0
```

| Field Name | Data Type | Encoding | Allowed Values | Semantics & Supervisor Reaction |
|---|---|---|---|---|
| `daemon` | `string` | ASCII / UTF-8 | Absolute path | Absolute filesystem path to the executable binary or shell script to be managed `[CONFIRMED]`. |
| `port` | `uint16` | Decimal | `0` to `65535` | TCP port number that the supervisor binds to for on-demand socket activation. Set to `0` when `auto=1` `[CONFIRMED]`. |
| `max` | `uint8` | Decimal | `1` to `255` | Maximum number of concurrent process instances the supervisor is permitted to spawn simultaneously (typically `1`) `[CONFIRMED]`. |
| `auto` | `uint8` | Decimal | `0` or `1` | **Startup Policy Flag:**<br>• `1` (Immediate): The supervisor forks and executes the daemon immediately upon supervisor startup. The `port` and `max` fields are ignored `[CONFIRMED]`.<br>• `0` (On-Demand): The supervisor binds to `port` and listens for incoming TCP connection requests. When a client connects, the supervisor forks an instance of `daemon` to service the connection `[CONFIRMED]`. |

#### 2.4.2 Process Monitoring & Failure Recovery

The supervisor retains process group control over all spawned children:

- When an auto-started daemon terminates unexpectedly (non-zero exit code or uncaught signal), `svcd.exe` restarts the service; restart rate limits are not specified `[INFERRED]`.
- For on-demand socket daemons (e.g. `svc_exam` on port 22222), the supervisor accepts the TCP connection and launches the target program with the connected socket attached as its standard input and output `[INFERRED]`.

---

## 3. Process Inventory

The table below catalogs every runtime userland process active on the controller, documenting its executable path, privilege level, supervisory parent, primary responsibility, and clean-room interoperability requirement tier:

- **Critical:** Essential for system operation, network staging, project activation, or basic automation execution.
- **Optional:** Specific to specialized peripheral hardware (e.g. streaming audio players, legacy iPod docks) or cloud remote access portals.
- **Auxiliary:** System logging, hardware health monitoring, or console administration.

| Daemon Name | Executable Path | User | Supervised By | Primary Functional Responsibility | Interoperability Tier |
|---|---|---|---|---|---|
| `init` | `/sbin/init [5]` | `root` | Kernel (PID 1) | System initialization, runlevel state management, and orphaned process reaping. | **Critical** `[CONFIRMED]` |
| `udevd` | `/sbin/udevd -d` | `root` | `rcS.d/S04udev` | Kernel device event manager; dynamic node population under `/dev/`. | **Critical** `[CONFIRMED]` |
| `udhcpc` | `/sbin/udhcpc` | `root` | `rc5.d/S01networking` | BusyBox DHCP client managing network lease and IP allocation on `eth0`. | **Critical** `[CONFIRMED]` |
| `dropbear` | `/usr/sbin/dropbear` | `root` | `rc5.d/S10dropbear` | SSH daemon listening on TCP port 22 for console diagnostics and remote administration. | **Optional** (Console) `[CONFIRMED]` |
| `syslogd` | `/sbin/syslogd` | `root` | `rc5.d/S20syslog` | System logging daemon recording events to `/var/log/messages`. | **Auxiliary** `[CONFIRMED]` |
| `klogd` | `/sbin/klogd` | `root` | `rc5.d/S20syslog` | Kernel log daemon capturing kernel printk buffer events. | **Auxiliary** `[CONFIRMED]` |
| `dhcpmngd` | `/remote/App/dhcpmngd` | `root` | `rc5.d/S60RCMain.sh` | Network interface monitor, DHCP state tracking, and interface renewal. | **Critical** `[CONFIRMED]` |
| `provisioningd` | `/remote/App/provisioningd` | `root` | `rc5.d/S60RCMain.sh` | Controller provisioning agent listening on UDP port 61510 for discovery probes and staging bootstrap. | **Critical** `[CONFIRMED]` |
| `svcd.exe` | `/remote/App/svcd.exe` | `root` | `rc5.d/S60RCMain.sh` | Service supervisor executing policies from `/remote/App/service.ini`; manages port 22222 socket activation. | **Critical** `[CONFIRMED]` |
| `early_init` | `/remote/App/early_init` | `root` | `svcd.exe` | One-shot initialization utility; stops default busybox FTP and spawns `vsftpd` with staging configuration. | **Critical** `[CONFIRMED]` |
| `vsftpd` | `/remote/Store/App/vsftpd` | `root` | `early_init` | Production FTP daemon listening on TCP port 21 accepting project archive uploads to `/remote/Store/updateroot`. | **Critical** `[CONFIRMED]` |
| `mosquitto` | `/remote/Store/App/mosquitto` | `root` | `mqttinit.sh` (`svcd.exe`) | Local MQTT broker listening on TCP port 1883 for inter-process message and event distribution. | **Critical** `[CONFIRMED]` |
| `mdnsd` | `/remote/App/mdnsd` | `nobody` | `svcd.exe` | Multicast DNS responder on UDP port 5353. The verified firmware does not advertise `_urc._tcp`; clients must not rely on that service for controller discovery. | **Auxiliary** `[CONFIRMED]` |
| `deviceguard` | `/remote/App/deviceguard` | `root` | `svcd.exe` | Device license verification, room state management, and local REST API service on TCP port 54211. | **Critical** `[CONFIRMED]` |
| `devmngd.exe` | `/remote/App/devmngd.exe` | `root` | `svcd.exe` | Hardware device manager handling serial ports, IR routing, relay states, and voltage sensor inputs. | **Critical** `[CONFIRMED]` |
| `BS.exe` | `/remote/App/BS.exe` | `root` | `rc5.d/S60RCMain.sh` | Core automation engine; macro scheduler, device driver host, and configuration activation listener (TCP 51510). | **Critical** `[CONFIRMED]` |
| `rui_svrd_launcher.exe` | `/remote/App/rui_svrd_launcher.exe` | `root` | `svcd.exe` | Spawner and lifecycle controller for user interface rendering server instances (TCP 51531). | **Critical** (UI) `[CONFIRMED]` |
| `rui_svrd.exe` | `/remote/Store/App/rui_svrd.exe` | `root` | `rui_svrd_launcher.exe` | Remote user interface daemon serving graphics, menus, and button events to touchscreens (TCP 51530/54501). | **Critical** (UI) `[CONFIRMED]` |
| `rmsmcsman.exe` | `/remote/App/rmsmcsman.exe` | `root` | `svcd.exe` | Cloud management agent; maintains SIP/STUN tunnel to `urcrms.com` for off-site dealer maintenance. | **Optional** (Cloud) `[CONFIRMED]` |
| `rmslocalsvrd.exe` | `/remote/App/rmslocalsvrd.exe` | `root` | `svcd.exe` | Local HTTP staging server for firmware updates and local management proxy (TCP 51520). | **Optional** (Cloud) `[CONFIRMED]` |
| `logmanager.exe` | `/remote/App/logmanager.exe` | `root` | `svcd.exe` | Centralized application logging manager collecting system debug streams over UDP port 51535. | **Auxiliary** `[CONFIRMED]` |
| `syswatch.exe` | `/remote/App/syswatch.exe` | `root` | `svcd.exe` | Software watchdog monitoring daemon health and performing system reboots upon unrecoverable faults. | **Auxiliary** `[CONFIRMED]` |
| `rarpd` | `/remote/App/rarpd` | `root` | `svcd.exe` | Reverse ARP responder providing MAC-to-IP resolution for legacy hardware clients. | **Optional** (Legacy) `[CONFIRMED]` |
| `snpd.exe` | `/remote/App/snpd.exe` | `root` | `svcd.exe` | Streaming Network Player coordinator managing audio playback sessions (TCP ports 56030, 56031). | **Optional** (Audio) `[CONFIRMED]` |
| `snpd_mrx10.exe` | `/remote/App/snpd_mrx10.exe` | `root` | `svcd.exe` | Streaming player coordination bridge for legacy MRX-10 sub-base stations (UDP port 49850). | **Optional** (Audio) `[CONFIRMED]` |
| `snp2d.exe` | `/remote/Store/App/snp2d.exe` | `root` | Dynamic (spawned by the audio-player manager) | Secondary network player audio streaming daemon (TCP ports 53000, 53001). | **Optional** (Audio) `[CONFIRMED]` |
| `ipodd_mx780.exe` | `/remote/App/ipodd_mx780.exe` | `root` | `svcd.exe` | Legacy iPod dock communication gateway for MX-780 handheld remote controls (UDP port 49749). | **Optional** (Legacy) `[CONFIRMED]` |
| `ipodd_mrx10.exe` | `/remote/App/ipodd_mrx10.exe` | `root` | `svcd.exe` | Legacy iPod dock communication gateway for MRX-10 sub-controllers (UDP port 49749). | **Optional** (Legacy) `[CONFIRMED]` |
| `xgated.exe` | `/remote/App/xgated.exe` | `root` | Dynamic (`BS.exe`) | Expansion gateway bridge coordinating communication across multiple networked sub-controllers (TCP 51200). | **Auxiliary** `[CONFIRMED]` |
| `getty` | `/sbin/getty` | `root` | `/etc/inittab` | Serial console terminal listener on UART port `ttymxc0` (115200 baud) and local console `tty1`. | **Auxiliary** `[CONFIRMED]` |

---

## 4. Network Sockets & Port Architecture

### 4.1 TCP Listening Sockets

The controller exposes 27 distinct TCP listening ports across all network interfaces (`0.0.0.0`) or restricted to local loopback (`127.0.0.1`):

| Port | Bind IP | Owning Daemon | Functional Role | Programmer Interaction & Purpose | Confidence |
|---|---|---|---|---|---|
| **21** | `0.0.0.0` | `vsftpd` | FTP configuration and staging server. | **Direct Staging:** Programmers connect via anonymous or authenticated FTP to upload project configurations, drivers, and UI assets into `/remote/Store/updateroot`. | `[CONFIRMED]` |
| **22** | `0.0.0.0` | `dropbear` | Secure shell (SSH) remote administration. | **Diagnostics / Console:** Secure command-line access for firmware debugging, log inspection, and troubleshooting. | `[CONFIRMED]` |
| **1883** | `0.0.0.0` | `mosquitto` | Local MQTT message broker. | **Event Monitoring:** Subscribing to controller status events, sensor triggers, and system state transitions over standard MQTT. | `[CONFIRMED]` |
| **5060** | `0.0.0.0` | `rmsmcsman.exe` | SIP signaling agent for remote cloud traversal. | **Cloud Tunnel:** Communicates with remote management cloud proxy for off-site NAT traversal; optional for local programming. | `[CONFIRMED]` |
| **22222** | `0.0.0.0` | `svcd.exe` | On-demand socket activation trigger (`svc_exam`). | **Diagnostic Trigger:** Test port managed by supervisor for on-demand inspection daemon invocation. | `[CONFIRMED]` |
| **46120** | `127.0.0.1` | `rui_svrd.exe` | Internal UI server loopback IPC socket. | **Internal IPC:** Loopback-only internal channel for the UI server; not intended for external clients. | `[CONFIRMED]` |
| **50252** | `0.0.0.0` | `deviceguard` | DeviceGuard authentication and pairing socket. | **Commissioning:** Controller-to-client identity verification and credential exchange during initial setup. | `[CONFIRMED]` |
| **50257** | `0.0.0.0` | `deviceguard` | DeviceGuard secure communication channel. | **Security Exchange:** Encrypted token validation and device certificate authorization. | `[CONFIRMED]` |
| **51200** | `0.0.0.0` | `xgated.exe` | Expansion gateway cross-connect listener. | **Multi-Controller Sync:** Synchronization channel when linking secondary expansion sub-controllers. | `[CONFIRMED]` |
| **51510** | `0.0.0.0` | `BS.exe` / `xgated.exe` | Base station control and staging activation listener. | **Direct Programming Trigger:** Primary command port. Programmers send binary trigger frames to initiate checksum validation, manifest unpack, and runtime engine reload. | `[CONFIRMED]` |
| **51511** | `0.0.0.0` | `BS.exe` | Secondary base station control and diagnostic listener. | **Project Synchronization:** Project manifests, graphics, and database archives are synchronized via anonymous FTP (port 21) or SFTP (port 22); port 51511 serves as a legacy secondary control listener. | `[CONFIRMED]` |
| **51520** | `0.0.0.0` | `rmslocalsvrd.exe` | Local binary staging and version query server. | **Staging Protocol:** Proprietary binary protocol endpoint (`0x2010` version query, `0x2070` pre-download) used by programming tools and mobile clients to query models and stage release packages. | `[CONFIRMED]` |
| **51530** | `0.0.0.0` | `rui_svrd.exe` | Remote User Interface (RUI) master listener (Mode 2). | **UI Client Control:** Primary communication channel for touchscreens and mobile apps requesting UI screen definitions. | `[CONFIRMED]` |
| **51531** | `0.0.0.0` | `rui_svrd_launcher.exe` | RUI launcher supervision and IPC channel. | **Internal IPC:** Process manager endpoint used by `rui_svrd_launcher.exe` to manage child RUI server lifecycles. | `[CONFIRMED]` |
| **51533** | `0.0.0.0` | `rui_svrd.exe` | RUI client event and status feed socket. | **UI Event Stream:** Delivers real-time button state updates, volume feedback, and text strings to connected touchscreens. | `[CONFIRMED]` |
| **51540** | `0.0.0.0` | `rmsmcsman.exe` | Cloud management synchronization listener. | **Cloud Sync:** Remote management client synchronization endpoint. | `[CONFIRMED]` |
| **53000** | `0.0.0.0` | `devmngd.exe` / `snp2d.exe` | Device manager hardware status query socket. | **Peripheral Audio Query:** Queries peripheral audio distribution devices (HDA amplifiers) via JSON-RPC. Not a general-purpose raw I/O port for LAN clients. | `[CONFIRMED]` |
| **53001** | `0.0.0.0` | `devmngd.exe` / `snp2d.exe` | Device manager command dispatch socket. | **Peripheral Audio Control:** Transmits JSON configuration and streaming commands to peripheral HDA amplifiers. Physical relays, sensors, and RS-232 ports on the controller are executed via programmed macros in `BS.exe`. | `[CONFIRMED]` |
| **54211** | `0.0.0.0` | `deviceguard` | DeviceGuard internal HTTP REST API. | **Internal REST API:** Exposes endpoints `/deviceguard_api/settcroominfo` and `/deviceguard_api/updatebsinfo` used by `BS.exe`. | `[CONFIRMED]` |
| **54215** | `0.0.0.0` | `deviceguard` | DeviceGuard auxiliary licensing port. | **License Validation:** Validates feature licenses and room seat allocations. | `[CONFIRMED]` |
| **54216** | `0.0.0.0` | `deviceguard` | DeviceGuard certificate validation socket. | **Security Verification:** Inter-controller mutual authentication channel. | `[CONFIRMED]` |
| **54218** | `0.0.0.0` | `deviceguard` | DeviceGuard secure inter-device sync socket. | **Peer Sync:** Encrypted peer-to-peer communication between redundant or sub-station controllers. | `[CONFIRMED]` |
| **54501** | `0.0.0.0` | `rui_svrd.exe` | Dynamic RUI client session socket (instance `cport`). | **Active UI Session:** Dedicated bidirectional socket assigned to a specific connected touchscreen instance. | `[CONFIRMED]` |
| **56030** | `0.0.0.0` | `snpd.exe` | Streaming Network Player (SNP-1) control listener. | **Media Player Control:** Transmits transport controls (play, pause, skip) and playlist management commands to SNP units. | `[CONFIRMED]` |
| **56031** | `0.0.0.0` | `snpd.exe` | SNP player metadata and track progress event feed. | **Media Metadata Feed:** Delivers track titles, album art metadata, and playback progress to UI clients. | `[CONFIRMED]` |
| **58599** | `0.0.0.0` | `rui_svrd.exe` | Dynamic RUI asset and graphics delivery port. | **Asset Delivery:** Streams bitmap graphics, icons, and fonts to connected UI client hardware. | `[CONFIRMED]` |
| **59410** | `0.0.0.0` | `rui_svrd.exe` | Dynamic RUI macro and action event delivery port. | **Action Dispatch:** Transmits button press event triggers from UI clients into the macro execution pipeline. | `[CONFIRMED]` |

### 4.2 UDP Sockets

The controller binds 17 UDP sockets for local network discovery, subsystem heartbeats, sensor broadcasts, and internal event loops:

| Port | Bind IP | Owning Daemon | Functional Role | Programmer Interaction & Purpose | Confidence |
|---|---|---|---|---|---|
| **5353** | `0.0.0.0` | `mdnsd` | Multicast DNS responder daemon. | **Optional Name Service:** Handles mDNS traffic, but the verified firmware does not advertise `_urc._tcp`; active controller discovery uses UDP 61510. | `[CONFIRMED]` |
| **37912** | `0.0.0.0` | `rmslocalsvrd.exe` | RMS local network broadcast listener. | **Cloud Proxy Discovery:** Detects local RMS proxy instances and update staging servers on the subnet. | `[CONFIRMED]` |
| **43217** | `0.0.0.0` | `mdnsd` | Auxiliary mDNS query responder socket. | **Discovery Query:** Responds to unicast DNS queries from network commissioning clients. | `[CONFIRMED]` |
| **49749** | `0.0.0.0` | `ipodd_mx780.exe` / `ipodd_mrx10.exe` | Legacy iPod dock status broadcast listener. | **Legacy Dock Integration:** Listens for broadcast state changes from physical hardware iPod docking stations. | `[CONFIRMED]` |
| **49850** | `0.0.0.0` | `snpd_mrx10.exe` | Streaming player discovery and status listener. | **Audio Discovery:** Discovers network-attached SNP streaming players on the broadcast subnet. | `[CONFIRMED]` |
| **51535** | `0.0.0.0` | `logmanager.exe` | Application debug logging receiver and broadcast. | **Diagnostic Log Stream:** Captures formatted debug messages broadcast by secondary userland daemons. | `[CONFIRMED]` |
| **55170** | `0.0.0.0` | `BS.exe` / `xgated.exe` | Base station peer-to-peer heartbeat beacon. | **Controller Presence:** Broadcasts alive heartbeats every 1000ms to detect duplicate master controllers or online expansion units. | `[CONFIRMED]` |
| **57000** | `0.0.0.0` | `devmngd.exe` / `snp2d.exe` | Hardware device manager peripheral broadcast listener. | **Peripheral Listener:** Inbound subnet broadcast receiver for peripheral device announcements and voltage sensor edge events. Inner payload field layout is unratified. | `[INFERRED]` |
| **57002** | `0.0.0.0` | `BS.exe` / `xgated.exe` | Base station macro event notification receiver. | **Macro Event Listener:** Inbound subnet broadcast listener for network macro triggers, inter-station coordination, and paging announcements. Inner payload field layout is unratified. | `[INFERRED]` |
| **57008** | `0.0.0.0` | `rmsmcsman.exe` | RMS STUN keepalive and NAT traversal socket. | **NAT Traversal:** Transmits UDP keepalive packets to cloud STUN servers to preserve stateful firewall pinholes. | `[CONFIRMED]` |
| **57100** | `0.0.0.0` | `devmngd.exe` | Peripheral I/O discovery and ping socket. | **Peripheral Discovery:** Broadcasts discovery probes to find network-attached serial extenders and sensor modules. | `[CONFIRMED]` |
| **57355** | `127.0.0.1` | `BS.exe` | Internal base station loopback event channel. | **Internal Loopback:** Loopback-only internal channel; not intended for external clients. | `[CONFIRMED]` |
| **61510** | `0.0.0.0` | `provisioningd` | URC proprietary controller discovery broadcast. | **Direct Commissioning Probe:** Listens for UDP broadcast discovery frames from programming software; replies with controller model, MAC, and IP. | `[CONFIRMED]` |
| **61520** | `0.0.0.0` | `BS.exe` / `xgated.exe` | Base station client announcement channel. | **Client Announcement:** Notifies newly connected touchscreens of the controller master identity. | `[CONFIRMED]` |
| **61610** | `0.0.0.0` | `BS.exe` / `xgated.exe` | Sub-controller synchronization broadcast. | **Multi-Controller Sync:** Synchronizes distributed macro state and global variable tables across linked controllers. | `[CONFIRMED]` |
| **61611** | `0.0.0.0` | `BS.exe` | Base station secondary automation notification socket. | **Secondary Event Socket:** Secondary automation notification channel. Inner payload field layout is unratified. | `[INFERRED]` |
| **61620** | `0.0.0.0` | `deviceguard` | DeviceGuard pairing discovery broadcast socket. | **Security Discovery:** Discovers compatible secure client interfaces on the local subnet. | `[CONFIRMED]` |

### 4.3 Network Security Architecture & Exposure Model

The controller's network exposure model reflects an embedded device designed for isolated local area network operation:

1. **Unauthenticated Staging Services:** The FTP server (`vsftpd` on TCP port 21) accepts anonymous connections to write staging archives into `/remote/Store/updateroot`. The core activation port (TCP port 51510) accepts unauthenticated trigger frames from any host within the broadcast domain.
2. **Network Perimeter Protection:** Controllers MUST NOT be exposed directly to the public Internet without a stateful firewall, VPN gateway, or VLAN isolation. All programming, control, and staging protocols operate over cleartext or proprietary framed TCP/UDP connections.
3. **Loopback Binding Isolation:** Highly sensitive internal control interfaces (e.g. `rui_svrd.exe` coordination on port 46120 and internal `BS.exe` signaling on UDP 57355) are strictly bound to `127.0.0.1`, preventing off-device packet injection.

---

## 5. Inter-Process Communication (IPC) Architecture

The controller userland utilizes three primary IPC paradigms for coordinating state across autonomous processes:

```
+-----------------------------------------------------------------------------------+
|                              LOCAL IPC MECHANISMS                                 |
+-----------------------------------------------------------------------------------+
|  1. Loopback TCP/UDP Sockets (127.0.0.1)                                          |
|     - 127.0.0.1:54211 -> DeviceGuard REST API (settcroominfo, updatebsinfo)       |
|     - 127.0.0.1:46120 -> rui_svrd.exe internal coordination                       |
|     - 127.0.0.1:57355 -> BS.exe asynchronous task signal loop                     |
+-----------------------------------------------------------------------------------+
|  2. MQTT Message Bus (TCP 1883 via mosquitto)                                     |
|     - Topic: device/status/* -> Sensor state transitions, trigger events          |
|     - Topic: system/state/*  -> Subsystem heartbeats, room volume, macro state    |
+-----------------------------------------------------------------------------------+
|  3. Volatile File Descriptors (/tmp -> /var/volatile/tmp)                         |
|     - /tmp/dobj_<UID>.cmd   -> Dynamic RUI command descriptors                    |
|     - /tmp/diag_*.info      -> System diagnostic snapshots                        |
|     - /var/run/*.pid        -> Daemon process ID locks                            |
+-----------------------------------------------------------------------------------+
```

### 5.1 Local Loopback Sockets (127.0.0.1)

Processes utilize internal loopback sockets to execute synchronized Remote Procedure Calls (RPC) and event notifications:

1. **DeviceGuard Local REST API (`127.0.0.1:54211`):**
   - Managed by `deviceguard`.
   - Used extensively by the core engine (`BS.exe`) to register active system state, room configuration, and license parameters.
   - Known endpoints include:
     - `POST /deviceguard_api/settcroominfo`: Updates active room boundaries, user permissions, and zone groupings `[CONFIRMED]`.
     - `POST /deviceguard_api/updatebsinfo`: Reports base station firmware version, MAC address, and hardware serial identity `[CONFIRMED]`.
2. **RUI Server Coordination (`127.0.0.1:46120`):**
   - Managed by `rui_svrd.exe`.
   - Internal coordination channel between UI server instances; not intended for external clients `[CONFIRMED]`.
3. **Core Engine Event Loop (`127.0.0.1:57355`):**
   - Loopback UDP socket managed by `BS.exe`.
   - Internal loopback datagram channel for the automation engine; not intended for external clients `[CONFIRMED]`.

### 5.2 MQTT Message Bus (`mosquitto` on TCP 1883)

The local MQTT message broker (`mosquitto`) runs as a persistent service launched via `/remote/App/mqttinit.sh` using configuration `/remote/Store/App/mosquitto.conf` `[CONFIRMED]`.

- **Transport:** Standard MQTT 3.1.1 protocol listening on TCP port 1883 `[CONFIRMED]`.
- **Authentication:** None. The default broker configuration enforces `allow_anonymous true`; clients connect anonymously without username or password `[CONFIRMED]`.
- **Directionality & Role:** Outbound publish-subscribe telemetry distribution bus. The controller engine `BS.exe` operates strictly as an MQTT publisher; it does NOT subscribe to incoming command topics on TCP 1883. Publishing to MQTT topics does NOT control the controller or invoke macros `[CONFIRMED]`.
- **QoS & Retention Policy:**
  - All telemetry publishes with **QoS 0** `[CONFIRMED]`.
  - Retained messages were observed for `zone_status`, `hda/zone`, `hda/input`, and `hda/ampstatus` at QoS 0. A broker sends an existing retained value to a new subscriber, but a missing retained value is not a state snapshot and a retained value may be stale `[CONFIRMED]`.
  - Transient events need not be retained. Retention and initialization behavior for every topic and firmware version remains unverified.
  - No complete startup snapshot is guaranteed. If a topic has no retained value, its state is unknown until an update or an independently verified query provides it; this specification does not establish a general query to force a complete snapshot.
  - The shipped broker configuration enables no persistence, so the retained set contains only values published since the broker last started (normally boot) `[CONFIRMED]`.
  - Observation, one controller with one 24-zone HDA amplifier and three rooms, after multiple days of uptime `[CONFIRMED]`: `hda/input` (inputs 1–8) and `hda/ampstatus` (channels 1–16) were retained with time stamps about 25 s after boot. `hda/zone` was retained for zone `0` only and `zone_status` for one room only, both time-stamped at a runtime change days later. The other 23 zones and two rooms had no retained value. A 120 s subscription without retained messages received no publishes.

#### 5.2.1 Standard Telemetry Envelope

Every message published to the MQTT bus wraps its payload in a standard JSON envelope `[CONFIRMED]`:

```json
{
  "header": {
    "version": "0.0.0.0",
    "seq": 0,
    "time_stamp": 1700000000,
    "tree": "tcsystem/global/room_event/zone_status/room_id_2"
  },
  "body": ...
}
```

| Envelope Field | JSON Type | Description |
|---|---|---|
| `header.version` | String | Observed `"0.0.0.0"` on every observed message `[CONFIRMED]`; no other value has been observed. |
| `header.seq` | Integer | Observed as `0` on every observed message, including startup publishes and runtime-change publishes days later; it is not a monotonically increasing counter in observed traffic. Whether it is ever non-zero is unresolved. Clients MUST accept `0` and MUST NOT rely on it for ordering or deduplication `[CONFIRMED]`. |
| `header.time_stamp` | Integer | Publication time in seconds. Observed values were offset from the controller's UTC system clock by its local timezone offset, i.e. local wall-clock time encoded as seconds since 1970. Clients MUST NOT assume true UTC `[CONFIRMED]`. |
| `header.tree` | String | Target MQTT topic string representing the message tree `[CONFIRMED]`. |
| `body` | Object | Telemetry payload specific to the topic category; every observed body was one JSON object `[CONFIRMED]`. |

#### 5.2.2 Topic Grammar & Body Schemas

Telemetry topics append explicit hierarchical identifiers to discriminate event sources:

##### 1. Room Zone Status (`tcsystem/global/room_event/zone_status/room_id_{room_id}`)
- **Topic Grammar:** `tcsystem/global/room_event/zone_status/room_id_{room_id}` where `{room_id}` is the physical room ID (`data.rooms[].id` in `core_model.json`).
- **Retain:** `1` (QoS 0) `[CONFIRMED]`.
- **Body Schema:** A single JSON object was observed, with room identity in the topic and no `room_id` field in the body:

```json
{"zone_power": 1, "cur_vol": 45, "min_vol": 0, "max_vol": 100, "mute": 0, "fixed": 0}
```

Clients MUST NOT require an array or a `room_id` body field. Other variants have not been ruled out.

| Body Field | Type | Range / Format | Description |
|---|---|---|---|
| Room identity | Topic suffix | `room_id_{room_id}` | Identifies the room in the observed message; `room_id` was absent from the body `[CONFIRMED]`. |
| `zone_power` | Integer | `0` or `1` | Power state: `0` = Off, `1` = On `[CONFIRMED]`. |
| `cur_vol` | Integer | Device-dependent | Current volume value; units and general range are not established `[CONFIRMED]`. |
| `min_vol` | Integer | Device-dependent | Minimum volume bound; units and general range are not established `[CONFIRMED]`. |
| `max_vol` | Integer | Device-dependent | Maximum volume bound; units and general range are not established `[CONFIRMED]`. |
| `mute` | Integer | `0` or `1` | Mute status: `0` = Unmuted, `1` = Muted `[CONFIRMED]`. |
| `fixed` | Integer | `0` or `1` | Output mode: `0` = Variable volume, `1` = Fixed line output `[CONFIRMED]`. |

##### 2. HDA Matrix Zone Telemetry (`tcsystem/global/hda/zone/{hda}/{zone}/`)
- **Topic Grammar:** `tcsystem/global/hda/zone/{hda}/{zone}/`; the trailing slash was present in the observed topic. In the one observed topic `hda/zone/2/0/`, `{hda}` equaled the HDA amplifier's `con_devs[].id` (`2`) in `mcs.dat` and `{zone}` was `0`, which is the first `zones[].idx` of that amplifier; the body `zoneid` was `"z1lr"`. A general mapping is not established beyond this observation.
- **Retain:** A retained QoS 0 message was observed.
- **Body Schema:** The observed body is one object. Example with synthetic labels:

```json
{
  "amp": {"peak": "normal", "thermal": "normal"},
  "balance": "0", "bass": "0", "treble": "0", "lipsync": "0",
  "ducking": {"status": "off"}, "eq": "Off", "eqcustomname": "Custom",
  "hdaid": "2", "inputdevname": "Source", "inputhdaid": "2", "inputhdaport": "1",
  "mute": "off", "power": "on", "roomid": "2", "roomindex": "2",
  "roomname": "Room", "stereo": "stereo",
  "volume": {"level": "45", "max": "100", "min": "0", "type": "variable"},
  "zoneid": "z1lr"
}
```

In the observed topic `hda/zone/2/0/`, `roomindex` was `"2"`, not `0`; it equaled the body `roomid` and the bound room's `data.rooms[].id`. Clients MUST NOT equate the topic zone component with `roomindex`; the meaning of `roomindex` is unresolved. `hdaid` equaled the topic `{hda}` and the amplifier's `con_devs[].id` in the observation, but clients MUST NOT assume that equality generally. Every scalar in the observed body was a JSON string, including identifiers. Additional fields and other representations MAY occur.

| Body Field | Type | Allowed Values | Description |
|---|---|---|---|
| `volume.type` | String | `"variable"`, `"fixed"` | Output volume mode `[CONFIRMED]`. |
| `volume.min`, `volume.max`, `volume.level` | String | Numeric text | Observed bounds and level; units and all supported ranges remain unverified. |
| `mute`, `power` | String | `"on"`, `"off"` observed | Observed state. |
| `amp.thermal`, `amp.peak` | String | `"normal"` observed | Amplifier health indicators nested under `amp`. |
| `roomid`, `roomindex`, `hdaid`, `zoneid` | String | Text | Identifiers in the observed body; their full mapping is unresolved. |
| Other fields | Various | See example | Clients SHOULD tolerate additional fields and unknown values. |

For a room whose audio is an HDA zone, one runtime change produced both `room_event/zone_status/room_id_2` and `hda/zone/2/0/` with the same `header.time_stamp` and corresponding values (`cur_vol` `81` / `volume.level` `"81"`, power on, unmuted) `[CONFIRMED]`. They are two views of the same change: `zone_status` is the room-level view keyed by room ID; `hda/zone` adds amplifier-zone detail (tone, source, room binding, amplifier health). Clients modelling rooms SHOULD use `zone_status`; clients needing amplifier-zone detail SHOULD use `hda/zone`. No precedence rule is specified, and this does not establish that both are always published or synchronized. Treat each as a potentially stale observation.

##### 3. System Variable Updates (`tcsystem/global/variable_status/variable_id_{variable_id}`)
- **Topic Grammar:** `tcsystem/global/variable_status/variable_id_{variable_id}` where `{variable_id}` is the 32-bit state variable index (`Table 8` in `DnData.dn`) `[CONFIRMED]`.
- **Retain:** `1` (QoS 0) `[CONFIRMED]`.
- **Body Schema:**

```json
{
  "variable_type": 0,
  "value": "1"
}
```

| Body Field | Type | Range / Format | Description |
|---|---|---|---|
| `variable_type` | Integer | `0`, `1`, `2` | Variable datatype: `0` = Boolean, `1` = Integer, `2` = String `[CONFIRMED]`. |
| `value` | String | String | String representation of current variable state `[CONFIRMED]`. |

##### 4. Room Link State (`tcsystem/global/room_event/link_status/link_id_{link_id}`)
- **Topic Grammar:** `tcsystem/global/room_event/link_status/link_id_{link_id}` where `{link_id}` is the synchronized audio link group ID `[CONFIRMED]`.
- **Retain:** `1` (QoS 0) `[CONFIRMED]`.
- **Body Schema:**

```json
{
  "link_id": 1,
  "roomid": 1,
  "roomname": "Living Room",
  "linked_rooms": [1, 2]
}
```

##### 5. HDA Input and Amplifier Status

- **Input:** `tcsystem/global/hda/input/{hda}/{input}/` was received retained at QoS 0 for inputs `1`–`8`, published at startup. Its observed body contained only three strings, for example `{"audiosense":"not_sensing","hdaid":"2","inputport":"1"}`; `inputport` equaled the topic `{input}`. The preamplifier fields listed for this topic in earlier versions (`level`, `phantom`, `threshold`, and others) were not observed on it. Other variants are not excluded.
- **Amplifier:** `tcsystem/global/hda/ampstatus/{hda}/{zone}/` was received retained at QoS 0 for `{zone}` `1`–`16` on a 24-zone amplifier, published at startup. The observed body contained strings, for example `{"hdaid":"2","zoneid":"1","leftpeak":"normal","leftthermal":"normal","rightpeak":"normal","rightthermal":"normal"}`; `zoneid` equaled the topic `{zone}`. The controller publishes this topic, so clients MAY consume it but MUST NOT require it. The mapping between its `{zone}` numbers and `hda/zone` zone components is unresolved; clients MUST NOT assume `ampstatus zone = hda/zone zone + 1`.
- Both use the standard envelope. Do not infer that all HDA models, firmware versions, or input types have identical fields or retention behavior.

##### 6. Intercom & Door Station Events
- **Camera Popup:** `tcsystem/global/camera_popup` (`retain: 0`) with body containing `camera_id`, `client_id`, `popup_status` `[CONFIRMED]`.
- **Door Station Event:** `tcsystem/enc/global/door_station_event` (`retain: 0`) carrying encrypted or tokenized intercom call events `[CONFIRMED]`.
- **Paging Announcement:** `tcsystem/global/hda/announce/{announce_id}/` (`retain: 0`) carrying ducking activation and audio chime events `[CONFIRMED]`.
### 5.3 Volatile Staging & Command Descriptors (`/tmp`)

The `/tmp` directory is mounted on a volatile `tmpfs` RAM disk (`/var/volatile/tmp`), ensuring that ephemeral command files do not cause flash write exhaustion on the eMMC:

1. **Dynamic UI Command Files (`/tmp/dobj_*.cmd`):**
   - Passed via command-line arguments when spawning client UI instances:
     ```bash
     /remote/Store/App/rui_svrd.exe 0 uid=11 mode=1 cport=54501 cmd=/tmp/dobj_11.cmd
     ```
   - Encodes binary layout configurations, dynamic control bindings, and screen resolution parameters for connected client touchscreens `[CONFIRMED]`.
2. **Diagnostic & Status Files:**
   - `/tmp/validip`: Temporary buffer used by `mrx-init-post.sh` to validate the syntax of static IP configurations before activating network interfaces `[CONFIRMED]`.
   - `/tmp/diag_device_info.info`: Real-time system hardware summary containing kernel uptime, MAC address, FPGA revision, and active peripheral counts `[CONFIRMED]`.
   - `/tmp/rms_status.info`: Runtime status and cloud connection state recorded by `rmsmcsman.exe` `[CONFIRMED]`.

---

## 6. Cloud & External Service Dependencies

### 6.1 Cloud Endpoints Inventory

The controller communicates with the following external cloud endpoints for remote dealer management, cloud backup, licensing, and weather telemetry:

| Hostname / Endpoint | Transport & Port | Managing Daemon | Functional Role in Cloud Ecosystem | Standalone LAN Interoperability Requirement |
|---|---|---|---|---|
| `www.urcrms.com` | HTTPS (TCP 443) | `BS.exe`, `rmsmcsman.exe` | Remote Management System (RMS) registration portal, dealer remote access claiming, and network time synchronization (`/service_api/tc/get_utc`). | **OPTIONAL.** Standalone LAN systems operate without cloud registration. System time synchronizes via local hardware RTC (`/dev/rtc0`) or standard LAN NTP `[CONFIRMED]`. |
| `u2-api.urcnet.com` | HTTPS (TCP 443) | `BS.exe`, `rmsmcsman.exe` | IoT device credential management and remote system telemetry reporting (`/v1/tc/bs/status/api`). | **OPTIONAL.** Cloud IoT service gateway; local device drivers communicate directly over LAN/RS-232 without cloud routing `[CONFIRMED]`. |
| `developer.urcmcs.com` | HTTP / HTTPS | `rmsmcsman.exe` | Dealer developer portal, driver download proxy, and remote diagnostic log ingestion. | **OPTIONAL.** Dealer maintenance only; local driver staging occurs via local FTP (port 21) `[CONFIRMED]`. |
| `filesvr1.urcmcs.com` | HTTP (TCP 80) | `rmsmcsman.exe` | Cloud file manager for offsite project backup, system restore archives, and template synchronization. | **OPTIONAL.** Local project models and configurations reside permanently in `/remote/Store/Prg` `[CONFIRMED]`. |
| `apikit.urcmcs.com` | HTTPS (TCP 443) | `BS.exe` | License server token request and feature activation API. | **OPTIONAL** for LAN operation. Feature licensing and seat allocation are outside the scope of this specification. |
| `weatherservice.urcmx.com` | HTTPS (TCP 443) | `BS.exe` | URC weather forecast and condition proxy service. | **OPTIONAL.** Feeds weather data to UI weather widgets; non-essential for automation macros or device control `[CONFIRMED]`. |
| `wlog.urcmx.com` | HTTPS (TCP 443) | `BS.exe` | Remote error logging and system crash reporting telemetry. | **OPTIONAL.** Telemetry only; system errors are logged locally to `/var/log/messages` `[CONFIRMED]`. |
| `gcm-http.googleapis.com`<br>`fcm.googleapis.com` | HTTPS (TCP 443) | `BS.exe` | Google / Firebase Cloud Messaging gateways for mobile push notifications. | **OPTIONAL.** Off-LAN mobile push notifications only; local touchscreens and mobile clients receive real-time updates directly via TCP port 51530 `[CONFIRMED]`. |
| `mail.urcrms.com` | TCP 25 / 587 (SMTP) | `BS.exe` | URC hosted mail relay for system alert macros. | **OPTIONAL.** System alert macro email delivery; fails if offline unless routed through a local LAN SMTP gateway `[CONFIRMED]`. |
| a fixed IP address (TCP 5061) | TLS / SIP (TCP 5061) | `rmsmcsman.exe` | Remote Management System SIP/STUN relay server for off-site firewall traversal. | **OPTIONAL.** Off-site dealer tunnel only. Completely inactive and unneeded on closed or isolated LANs `[CONFIRMED]`. |

### 6.2 Standalone LAN Operational Guarantee

The controller does not require cloud connectivity, remote vendor servers, or active Internet access for core automation and programming functions:

1. **Autonomous Execution:** The primary automation engine (`BS.exe`), physical hardware manager (`devmngd.exe`), co-processor FPGA logic, and UI server (`rui_svrd.exe`) execute entirely out of local flash storage (`/remote/Store`) and system memory.
2. **Local Staging & Activation:** Programming software interacts strictly through local LAN protocols:
   - Controller discovery via UDP broadcast (port 61510) and mDNS (port 5353).
   - Project archive transfer via anonymous/authenticated FTP (port 21) into `/remote/Store/updateroot`.
   - Project activation and engine reload via local framed TCP command packets to port 51510.
3. **Local Clock Preservation:** Although the controller attempts to query `www.urcrms.com/service_api/tc/get_utc` for time updates, failure to reach this endpoint does not prevent operation; the system maintains clock accuracy using its onboard hardware real-time clock (RTC) battery and standard `hwclock` utilities.
4. **Absence of Autonomous Firmware Polling:** The controller contains no background cron jobs, autonomous update daemons, or scheduled polling tasks that contact vendor cloud servers to check for or download firmware updates. The cloud management agent can fetch files on demand from the file service (see §6.1) into `/updateroot/direct_download`; this is interactive and dealer-driven via the remote RMS portal.
5. **WAN-Severed Subsystem Boundaries:** When WAN connectivity is completely severed (or when deployed on an air-gapped VLAN), only the following cloud-dependent ancillary services fail:
   - Off-site dealer remote maintenance and reverse signaling tunnels (RMS via SIP port 5061).
   - Live weather forecasts on touchscreen and remote widgets (`weatherservice.urcmx.com`).
   - Cloud license-service interactions (`apikit.urcmcs.com`); see §6.1.
   - Off-LAN mobile push notifications (`gcm-http.googleapis.com`, `fcm.googleapis.com`).
   - System alert emails delivered via the default URC cloud relay (`mail.urcrms.com`).

   All local macro processing, sensor triggers, relay/IR/serial hardware control, two-way driver execution, local mobile/touchscreen UI rendering, and local programming updates remain 100% operational.
6. **Network Isolation Requirements:** The cloud management agent contacts its relay by a fixed IP address rather than a DNS name, so DNS filtering alone does not block it. To keep the controller off the Internet, use default-deny WAN egress filtering or a subnet with no default gateway. Because local staging ports (FTP port 21, query port 51520, and reload trigger ports 51510/61520) lack authentication, restrict LAN access to authorized programmer hosts.

---

## 7. Test Vectors & Conformance Verification

To validate that an independent clean-room programming tool or emulator complies with this system overview, implementations MUST satisfy the following test vectors:

### 7.1 Test Vector 1: Core Staging Port Reachability & Service Identification

An implementer validating a live controller on a local subnet (e.g. `192.168.1.0/24`) or an emulated target MUST verify the following socket handshake and banner characteristics:

#### FTP Staging Service (TCP Port 21)
- **Action:** Open standard TCP connection to port 21.
- **Expected Banner:** Receives standard FTP banner containing `220` status code (e.g. `220 (vsFTPd 2.3.5)` or compatible response) `[CONFIRMED]`.
- **Expected Authentication:** Accepts `USER anonymous` with any arbitrary email string or `PASS anonymous`, establishing access to `/remote/Store` `[CONFIRMED]`.

#### Primary Base Station Control Listener (TCP Port 51510)
- **Action:** Open raw TCP socket connection to port 51510.
- **Expected Behavior:** Connection is accepted immediately (`SYN/ACK`); socket remains open awaiting binary control frames without premature termination `[CONFIRMED]`.

#### Discovery Broadcast Responder (UDP Port 61510)
- **Action:** Transmit the 5-byte little-endian discovery probe from `spec/50-discovery.md` to UDP 61510.
- **Expected Response:** The controller transmits a 55-byte UDP reply to the request's source port. The 5-byte request receives a broadcast reply; the exact 9-byte form supplies the unicast reply address `[CONFIRMED]`.

### 7.2 Test Vector 2: `service.ini` Parser Compliance

A clean-room configuration utility parsing `/remote/App/service.ini` MUST produce identical daemon execution tables when supplied with the canonical test configuration:

```ini
# Canonical Test Configuration
/remote/App/logmanager.exe          0        1        1
/remote/App/svc_exam                22222    1        0
```

- **Directive 1 Processing:**
  - `daemon`: `/remote/App/logmanager.exe`
  - `port`: `0` (ignored)
  - `max`: `1` (ignored)
  - `auto`: `1` -> Categorized as **Immediate Background Launch**. Spawns at supervisor initialization `[CONFIRMED]`.
- **Directive 2 Processing:**
  - `daemon`: `/remote/App/svc_exam`
  - `port`: `22222` -> Categorized as **Socket Activation Listener**. Binds TCP port 22222 `[CONFIRMED]`.
  - `max`: `1` -> Enforces maximum of 1 active child instance `[CONFIRMED]`.
  - `auto`: `0` -> Spawn deferred until inbound client connection on port 22222 `[CONFIRMED]`.

---

## 8. Open Items

The following architectural items require further empirical investigation and will be resolved in subsequent specification updates:

1. **Dynamic UI Command Serialization (`/tmp/dobj_*.cmd`):**
   - *Status:* Open.
   - *Problem Statement:* Determine the exact binary structure, header layout, and argument serialization used in `/tmp/dobj_<UID>.cmd` files passed to `rui_svrd.exe`.
   - *Resolution Plan:* Capture multiple dynamic command files from the running controller during active touchscreen pairing sessions and compare them.
2. **DeviceGuard Local REST Endpoint Schemas (Port 54211):**
   - *Status:* Open.
   - *Problem Statement:* Document the complete JSON schema for `/deviceguard_api/settcroominfo` and `/deviceguard_api/updatebsinfo`.
   - *Resolution Plan:* Trace local loopback HTTP payloads between `BS.exe` and `deviceguard` using loopback packet capture.
3. **On-Demand Socket Activation Handshake (Port 22222):**
   - *Status:* Open.
   - *Problem Statement:* Determine the expected client stimulus frame required to interact with `/remote/App/svc_exam` when triggered on TCP port 22222.
   - *Resolution Plan:* Connect to port 22222 on a test controller, observe process spawning in `ps`, and capture protocol response.
