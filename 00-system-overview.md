---
title: "System Overview & Process Architecture"
section: "00-system-overview"
version: "0.2.0"
status: draft
type: Architecture
confidence: confirmed
circumvention: false
open_items:
  - "Empirically decode the binary serialization structure of /tmp/dobj_*.cmd dynamic UI command files."
  - "Map full JSON request/response schema for DeviceGuard REST endpoints on 127.0.0.1:54211."
  - "Verify trigger packet layout for on-demand socket activation daemon svc_exam on TCP port 22222."
---

# System Overview & Process Architecture

> **LEGAL NOTICE & NOMINATIVE FAIR USE DISCLAIMER:**  
> This document is part of an independent, clean-room technical specification authored solely to achieve device and network interoperability. This project is not affiliated with, sponsored by, or endorsed by Universal Remote Control, Inc. (URC). All product names, trademarks, and registered trademarks (including "URC", "Total Control", "MRX-8", "MRX-10", "MRX-15", "MRX-30", and "Accelerator") are the property of their respective owners, and are used here exclusively for nominative identification and technical compatibility purposes under 15 U.S.C. § 1125 and controlling fair use doctrines.
## 1. System Overview

### 1.1 Architectural Scope & Interoperability Objective

URC Total Control network controllers (including the MRX-15, MRX-30, MRX-10, and MRX-8) serve as the central automation processors within residential and commercial automation deployments. The primary controller executes scheduling and logic engines, maintains device state models, coordinates bidirectional communication with IP peripherals, controls local physical I/O (infrared, RS-232 serial, dry contact relays, 12V voltage sensors), and serves remote user interface (RUI) definitions to native touchscreens, mobile handsets, and handheld remote controls.

The objective of this specification is to define the runtime architecture, initialization sequence, userland process topology, network listening sockets, inter-process communication (IPC) pathways, and external network dependencies of the controller. This document establishes the vendor-independent technical foundation necessary for third-party, open-source programming suites to discover controllers, stage automation projects, activate runtime configurations, and monitor system health without requiring proprietary vendor maintenance portals.

### 1.2 Hardware Platform & Operating Environment

The physical controller hardware and operating environment are characterized by the following specifications:

- **Main Application Processor:** NXP / Freescale i.MX6 DualLite application processor (`MCIMX6U5DVM10AC`), incorporating dual ARM Cortex-A9 MPCore execution cores operating at a nominal clock frequency of 792 MHz (configurable up to 996 MHz) with 32 KB instruction and data L1 caches and 512 KB unified L2 cache `[CONFIRMED]`.
- **System Memory:** 1 GB DDR3 SDRAM clocked at 400 MHz (800 MT/s) `[CONFIRMED]`.
- **Persistent Non-Volatile Storage:** 4 GB eMMC 4.41 flash memory module (`/dev/mmcblk3`) partitioned into system rootfs, data store, and factory recovery banks `[CONFIRMED]`.
- **Hardware Acceleration Co-Processor:** Altera (Intel) Cyclone IV E FPGA attached to the processor via an internal platform bus mapped to physical memory address `0x08000000` (`8000000.mrxfpga`). The FPGA co-processor handles real-time physical I/O acceleration, including infrared carrier modulation (38 kHz to 56 kHz), IR learning pulse capture, high-speed multi-channel UART FIFOs, sensor edge interrupt arbitration, and relay switching logic `[CONFIRMED]`.
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
| 2. Partition & Storage Binding (rcS.d: /dev/mmcblk3p3 -> /media, /media/remote -> |
|    /remote, /media/lib -> /var/lib, tmpfs -> /tmp)                                |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| 3. Hardware & Co-Processor Init (rc5.d: /sys/.../fpga_load <- MRX_30.rbf,        |
|    GPIO & LED timer setup, BSInit.exe environment validation)                     |
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

The technical architecture, runtime lifecycles, network listening sockets, and filesystem layouts specified in this document were discovered and verified through a multi-tier engineering methodology:

1. **Hardware Interface Probing & Serial Console Telemetry:**
   - Physical examination of the controller mainboard PCB identifying core hardware components: application processor, eMMC flash storage, volatile RAM, auxiliary FPGA, and Ethernet PHY `[CONFIRMED]`.
   - High-impedance bench verification of logic levels (3.3V DC) across passive hardware test points `[CONFIRMED]`.
   - Passive observation of serial console output to catalog bootloader milestones, kernel boot parameters, memory mappings, and storage partition layouts `[CONFIRMED]`.

2. **Storage Layout & Filesystem Architecture Mapping:**
   - Comprehensive mapping of storage partitions, Master Boot Record (MBR), bootloader sectors, operating system root partition, persistent storage, and recovery partition `[CONFIRMED]`.
   - Structural mapping of the `/remote/Store` application hierarchy with bidirectional cryptographic hashing (SHA-256) ensuring byte-level verification `[CONFIRMED]`.
   - Analysis of runtime initialization scripts (`/etc/init.d/`) to map system runlevels, partition mount arguments, and persistent bind mounts `[CONFIRMED]`.

3. **Interface & Protocol Model Synthesis:**
   - Systematic cataloging of listening ports, wire framing formats, and protocol command dispatchers `[CONFIRMED]`.
   - Functional analysis of execution flows, command structures, and state machines based on observable wire and storage artifacts `[CONFIRMED]`.
   - Dynamic correlation of socket endpoints with owning system services `[CONFIRMED]`.

4. **Dynamic Network Telemetry & Inode-to-Socket Mapping:**
   - Systematic socket-to-process correlation: mapping every listening TCP port (Table 4.1) and bound UDP socket (Table 4.2) by cross-referencing Linux kernel network tables (`/proc/net/tcp`, `/proc/net/udp`) with per-process file descriptor tables and command lines `[CONFIRMED]`.
   - Real-time packet capture on network interfaces (`eth0`, `lo`), streaming raw `.pcap` packet traces during discovery, staging, and activation cycles `[CONFIRMED]`.
   - Controlled active probing: injecting structured UDP discovery frames and TCP command datagrams to record controller response behaviors, timeouts, and error handling `[CONFIRMED]`.

5. **Controlled Service Emulation & Isolated Probing:**
   - Execution of isolated daemons within controlled execution environments to validate protocol parsing, status codes, and checksum evaluation `[CONFIRMED]`.
   - Synthetic probing against service endpoints (e.g., TCP 51520 staging queries) to confirm wire protocol behavior independently of physical hardware `[CONFIRMED]`.

6. **Filesystem State Tracking & Differential Analysis:**
   - Systematic filesystem state snapshotting of `/remote/Store` before, during, and after configuration staging, software reload triggers, and hardware reboots `[CONFIRMED]`.
   - Tracking installed databases (`DnData.dn`, `Link.dat`, `Event.dat`, `Alarm.dat`) and staging manifests in `/remote/Store/updateroot` to verify consumer daemon migrations and activation workflows `[CONFIRMED]`.

7. **Clean Model Synthesis & Round-Trip Validation:**
   - Independent reconstruction of binary databases, automation tables, and upload manifests using clean reference parsers and generators `[CONFIRMED]`.
   - Byte-exact round-trip verification against reference test fixtures, confirming byte preservation and structural layout fidelity for tested schemas `[CONFIRMED]`.
---

## 2. Boot Sequence & Service Lifecycle

### 2.1 Boot Sequence Phases

The boot sequence progresses deterministically from hardware power-on to full automation readiness across defined initialization phases:

| Phase | Init Mechanism | Component / Target | Primary Purpose | Dependencies | Execution Mode & Timing |
|---|---|---|---|---|---|
| **0: Kernel Boot** | Bootloader / U-Boot | Linux Kernel 4.1.42-fslc | Initializes CPU cores, MMU, device tree (`imx6dl-mrx.dtb`), mounts initial devtmpfs, and invokes `/sbin/init [5]`. | Hardware power | Synchronous `[CONFIRMED]` |
| **1: Sysinit Setup** | `/etc/inittab` | `/etc/init.d/rcS` | Mounts `/proc` and `/sys`, configures standard environment variables, and invokes `/etc/rcS.d/` startup scripts in numerical order. | Kernel init complete | Synchronous `[CONFIRMED]` |
| **2: Storage Bindings** | `rcS.d/S08mrx-init-pre.sh` | Flash Mount & Bind Service | Formats uninitialized partitions as ext4; mounts `/dev/mmcblk3p3` to `/media`; mounts recovery partition `/dev/mmcblk3p4` to `/backup` (read-only); bind-mounts `/media/remote` to `/remote`; bind-mounts `/media/lib` to `/var/lib`; populates default binaries if missing. | eMMC driver initialized | Synchronous (~1.5s) `[CONFIRMED]` |
| **3: Volatile Scratchpad** | `rcS.d/S37populate-volatile.sh` | RAM Disk Overlay | Mounts `tmpfs` at `/var/volatile` and symlinks `/tmp` to `/var/tmp` (`/var/volatile/tmp`) for scratch files and socket endpoints. | Filesystem mounts | Synchronous `[CONFIRMED]` |
| **4: Post-Mount Verification** | `rcS.d/S59mrx-init-post.sh` | Configuration Verifier | Verifies `/remote/Store/App` symlink `/remote/App`; checks `/etc/network/interfaces` syntax, falling back to DHCP if corrupted; stages default daemon binaries. | Storage bind mounts | Synchronous `[CONFIRMED]` |
| **5: Runlevel 5 Transition** | `/etc/inittab` | `/etc/init.d/rc 5` | Initiates multi-user networking runlevel; iterates scripts in `/etc/rc5.d/`. | `rcS` exit 0 | Synchronous `[CONFIRMED]` |
| **6: Network Stack** | `rc5.d/S01networking` | `udhcpc -i eth0` | Configures network loopback (`lo`) and Ethernet (`eth0`); requests IPv4 lease via DHCP; writes PID to `/var/run/udhcpc.eth0.pid`. | Network interface driver | Asynchronous background `[CONFIRMED]` |
| **7: System Diagnostics** | `rc5.d/S10dropbear`, `S20syslog` | `dropbear`, `syslogd`, `klogd` | Starts SSH server on TCP port 22 using host keys in `/var/lib/dropbear/`; starts syslog logger recording to `/var/log/messages`. | Loopback networking | Asynchronous background `[CONFIRMED]` |
| **8: FPGA Bitstream Load** | `rc5.d/S50RCInit.sh` | FPGA Firmware Loader | Verifies `/remote/Store/App/MRX_30.rbf`; copies bitstream to `/media/firmware`; loads bitstream into co-processor via sysfs platform interface; executes initialization check `/tmp/BSInit.exe`. | Storage and platform bus | Synchronous (~0.8s) `[CONFIRMED]` |
| **9: Base Daemons** | `rc5.d/S60RCMain.sh` | `dhcpmngd`, `provisioningd` | Spawns DHCP connection manager; spawns hardware provisioning daemon listening on UDP port 61510; starts initial FTP service. | Network address assigned | Asynchronous background `[CONFIRMED]` |
| **10: Daemon Supervisor** | `rc5.d/S60RCMain.sh` | `svcd.exe /remote/App/service.ini` | Spawns daemon supervisor process; parses `/remote/App/service.ini`; forks all daemons flagged for immediate start; binds port 22222 for on-demand activation. | Base daemons active | Asynchronous supervisor `[CONFIRMED]` |
| **11: Early Staging Prep** | `svcd.exe` | `early_init` | Terminates initial lightweight FTP server; forks production FTP server (`vsftpd`) on TCP port 21 accepting configuration uploads into `/remote/Store/updateroot`. | Supervisor launch | Synchronous one-shot `[CONFIRMED]` |
| **12: Automation Engine** | `rc5.d/S60RCMain.sh` | `BS.exe` | Launches core automation engine after a 2-second supervisor stabilization delay; opens control ports TCP 51510 and 51511; loads compiled project models from `/remote/Store/Prg`. | Supervisor and base daemons | Persistent daemon `[CONFIRMED]` |
| **13: UI Serving Tier** | `svcd.exe` | `rui_svrd_launcher.exe`, `rui_svrd.exe` | Spawns remote UI listener on TCP port 51530 and dynamic UI renderers on ports 54501/58599/59410 for connected client touchscreens. | Core engine running | Persistent daemon `[CONFIRMED]` |

### 2.2 Storage Binding & Filesystem Hierarchy

The storage subsystem relies on physical partition separation and Linux bind mounts to decouple the read-only operating system image from mutable user programming data and persistent configuration:

| Block Device / Path | Filesystem | Mount Target | Persistence | Description |
|---|---|---|---|---|
| `/dev/mmcblk3p1` | `ext4` | `/` | Read-Only (Operating System) | Base operating system root filesystem containing kernel modules, system shared libraries, and standard SysV init scripts `[CONFIRMED]`. |
| `/dev/mmcblk3p3` | `ext4` | `/media` | Read-Write (Persistent Storage) | Primary persistent user data partition on the eMMC flash. Automatically formatted as ext4 on boot if empty or corrupted `[CONFIRMED]`. |
| `/dev/mmcblk3p4` | `ext4` | `/backup` | Read-Only (Factory Golden) | Unmodified factory recovery storage containing baseline firmware archives and default runtime binaries `[CONFIRMED]`. |
| `/media/remote` | (Bind Mount) | `/remote` | Read-Write (Application Root) | Application runtime root directory. Contains active automation executables, driver packages, and user configuration databases `[CONFIRMED]`. |
| `/remote/Store` | Directory | `/remote/Store` | Read-Write (Data Store) | Persistent storage tree holding binary applications (`App/`), automation logic (`Prg/`), configuration data (`data/`), and staging files (`updateroot/`) `[CONFIRMED]`. |
| `/remote/Store/App` | Symlink | `/remote/App` | Read-Write (Execution Tree) | Convenience symbolic link pointing `/remote/App` directly to `/remote/Store/App` `[CONFIRMED]`. |
| `/remote/Store/updateroot` | Directory | `/remote/Store/updateroot` | Read-Write (Staging Root) | Target directory for anonymous and authenticated FTP project staging uploads from programming software `[CONFIRMED]`. |
| `/media/lib` | (Bind Mount) | `/var/lib` | Read-Write (State Directory) | Persistent storage for system state data, including persistent SSH host keys (`/var/lib/dropbear/dropbear_rsa_host_key`) `[CONFIRMED]`. |
| `tmpfs` | `tmpfs` | `/var/volatile` | Volatile (RAM Disk) | Ephemeral RAM disk for high-frequency logs, lock files, and inter-process command buffers. Symlinked via `/tmp -> /var/tmp` `[CONFIRMED]`. |

### 2.3 FPGA Bitstream Configuration Interface

The Altera Cyclone IV FPGA co-processor provides hardware timing and signal generation for physical controller interfaces. The bitstream loading mechanism MUST follow the sysfs kernel platform contract:

1. **Bitstream Firmware Artifact:** The bitstream binary is stored in raw binary format (`.rbf`) at `/remote/Store/App/MRX_30.rbf` `[CONFIRMED]`.
2. **Firmware Staging:** The startup script `/etc/init.d/RCInit.sh` verifies the existence of `/remote/Store/App/MRX_30.rbf` and copies it to `/media/firmware/MRX_30.rbf` `[CONFIRMED]`.
3. **Platform Device Trigger:** The sysfs platform device node `/sys/bus/platform/devices/8000000.mrxfpga/fpga_load` accepts the string filename of the bitstream relative to the system firmware search path `[CONFIRMED]`:
   ```bash
   echo MRX_30.rbf > /sys/bus/platform/devices/8000000.mrxfpga/fpga_load
   ```
4. **Kernel Platform Driver Action:** The kernel driver initiates passive serial or fast passive parallel configuration clocking to stream the bitstream into the FPGA SRAM configuration memory. Upon configuration assertion (`CONFIG_DONE` signal active high), the FPGA transitions to user mode, enabling internal memory-mapped registers for UART FIFOs, IR timing timers, and relay latches `[CONFIRMED]`.

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

- When an auto-started daemon terminates unexpectedly (non-zero exit code or uncaught signal), `svcd.exe` detects the child state change via `waitpid()` and restarts the service according to internal rate limits `[INFERRED]`.
- For on-demand socket daemons (e.g. `svc_exam` on port 22222), the supervisor accepts the incoming TCP connection, creates a connected socket descriptor, duplicates the socket to standard I/O (file descriptors 0 and 1), and invokes `execve()` on the target binary `[STATIC]`.

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
| `mdnsd` | `/remote/App/mdnsd` | `nobody` | `svcd.exe` | Multicast DNS daemon advertising controller presence (`_urc._tcp`) on UDP port 5353. | **Critical** `[CONFIRMED]` |
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
| `snp2d.exe` | `/remote/Store/App/snp2d.exe` | `root` | Dynamic (`snp2mng.so`) | Secondary network player audio streaming daemon (TCP ports 53000, 53001). | **Optional** (Audio) `[CONFIRMED]` |
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
| **46120** | `127.0.0.1` | `rui_svrd.exe` | Internal UI server loopback IPC socket. | **Internal IPC:** Loopback channel for inter-thread coordination within user interface rendering instances. | `[CONFIRMED]` |
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
| **5353** | `0.0.0.0` | `mdnsd` | Multicast DNS (mDNS) responder daemon. | **Controller Discovery:** Advertises `_urc._tcp` service records containing model name, firmware version, and controller UUID on multicast group `224.0.0.251`. | `[CONFIRMED]` |
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
| **57355** | `127.0.0.1` | `BS.exe` | Internal base station loopback event channel. | **Internal Loopback:** Used internally by `BS.exe` worker threads to signal the main event loop asynchronously. | `[CONFIRMED]` |
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
   - Coordinates dynamic UI rendering pipelines across multi-client instances, preventing font cache collisions and shared memory deadlocks `[CONFIRMED]`.
3. **Core Engine Event Loop (`127.0.0.1:57355`):**
   - Loopback UDP socket managed by `BS.exe`.
   - Provides an asynchronous interrupt mechanism allowing POSIX signal handlers and hardware worker threads to wake the primary macro scheduling loop `[CONFIRMED]`.

### 5.2 MQTT Message Bus (`mosquitto` on TCP 1883)

The local MQTT message broker (`mosquitto`) runs as a persistent service launched via `/remote/App/mqttinit.sh` using configuration `/remote/Store/App/mosquitto.conf` `[CONFIRMED]`.

- **Transport:** Standard MQTT 3.1.1 protocol listening on TCP port 1883 `[CONFIRMED]`.
- **Authentication:** None. The default broker configuration enforces `allow_anonymous true`; clients connect anonymously without username or password `[CONFIRMED]`.
- **Directionality & Role:** High-speed outbound publish-subscribe distribution bus. The controller engine `BS.exe` (`MosquittoMan.cpp`) operates strictly as a publisher (`DoTopicPublish`); it does not subscribe to incoming commands on TCP 1883. Publishing to MQTT topics does NOT control the controller or invoke macros `[CONFIRMED]`.
- **QoS & Retention:** Outbound telemetry publishes with QoS 0. Transient events use retain flag `0`, while static status objects (zone volume, EQ, variable values) are published with retain flag `1` `[CONFIRMED]`.
- **Telemetry Topics & Payloads:** Payloads are formatted as JSON text:
  - Zone volume & mute: `tcsystem/global/room_event/zone_status` and `tcsystem/global/hda/zone` (`zone_power`, `cur_vol`, `min_vol`, `max_vol`, `mute`, `treble`, `bass`, `balance`).
  - Room link state: `tcsystem/global/room_event/link_status` (`roomid`, `roomname`, `link_id`).
  - System variable modifications: `tcsystem/global/variable_status` (`variable_type`, `value`).
  - Multi-zone HDA matrix telemetry: `tcsystem/global/hda/input`, `tcsystem/global/hda/mic`, `tcsystem/global/hda/announce`, `tcsystem/global/hda/ampstatus` (`thermal`, `peaking`, `voltage`, `audiosense`).
  - Intercom & Security: `tcsystem/global/camera_popup`, `tcsystem/enc/global/door_station_event`.
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

The controller binaries contain hardcoded references to external cloud endpoints for remote dealer management, cloud backup, licensing, and weather telemetry:

| Hostname / Endpoint | Transport & Port | Managing Daemon | Functional Role in Cloud Ecosystem | Standalone LAN Interoperability Requirement |
|---|---|---|---|---|
| `www.urcrms.com` | HTTPS (TCP 443) | `BS.exe`, `rmsmcsman.exe` | Remote Management System (RMS) registration portal, dealer remote access claiming, and network time synchronization (`/service_api/tc/get_utc`). | **OPTIONAL.** Standalone LAN systems operate without cloud registration. System time synchronizes via local hardware RTC (`/dev/rtc0`) or standard LAN NTP `[CONFIRMED]`. |
| `u2-api.urcnet.com` | HTTPS (TCP 443) | `BS.exe`, `rmsmcsman.exe` | IoT device credential management and remote system telemetry reporting (`/v1/tc/bs/status/api`). | **OPTIONAL.** Cloud IoT service gateway; local device drivers communicate directly over LAN/RS-232 without cloud routing `[CONFIRMED]`. |
| `developer.urcmcs.com` | HTTP / HTTPS | `rmsmcsman.exe` | Dealer developer portal, driver download proxy, and remote diagnostic log ingestion. | **OPTIONAL.** Dealer maintenance only; local driver staging occurs via local FTP (port 21) `[CONFIRMED]`. |
| `filesvr1.urcmcs.com` | HTTP (TCP 80) | `rmsmcsman.exe` | Cloud file manager for offsite project backup, system restore archives, and template synchronization. | **OPTIONAL.** Local project models and configurations reside permanently in `/remote/Store/Prg` `[CONFIRMED]`. |
| `apikit.urcmcs.com` | HTTPS (TCP 443) | `BS.exe` | License server token request and feature activation API (`/LicenseeServer/api/*`). | **OPTIONAL.** Standard controller automation functions execute standalone without active cloud token leases `[CONFIRMED]`. |
| `weatherservice.urcmx.com` | HTTPS (TCP 443) | `BS.exe` | URC weather forecast and condition proxy service. | **OPTIONAL.** Feeds weather data to UI weather widgets; non-essential for automation macros or device control `[CONFIRMED]`. |
| `wlog.urcmx.com` | HTTPS (TCP 443) | `BS.exe` | Remote error logging and system crash reporting telemetry. | **OPTIONAL.** Telemetry only; system errors are logged locally to `/var/log/messages` `[CONFIRMED]`. |
| `gcm-http.googleapis.com`<br>`fcm.googleapis.com` | HTTPS (TCP 443) | `BS.exe` | Google / Firebase Cloud Messaging gateways for mobile push notifications. | **OPTIONAL.** Off-LAN mobile push notifications only; local touchscreens and mobile clients receive real-time updates directly via TCP port 51530 `[CONFIRMED]`. |
| `mail.urcrms.com` | TCP 25 / 587 (SMTP) | `BS.exe` | URC hosted mail relay for system alert macros. | **OPTIONAL.** System alert macro email delivery; fails if offline unless routed through a local LAN SMTP gateway `[CONFIRMED]`. |
| `20.121.207.6:5061` | TLS / SIP (TCP 5061) | `rmsmcsman.exe` | Remote Management System SIP/STUN relay server for off-site firewall traversal. | **OPTIONAL.** Off-site dealer tunnel only. Completely inactive and unneeded on closed or isolated LANs `[CONFIRMED]`. |

### 6.2 Standalone LAN Operational Guarantee

A critical architectural finding of this clean-room analysis is that **URC Total Control MRX network controllers do NOT require cloud connectivity, remote vendor servers, or active Internet access for core automation and programming functions**:

1. **Autonomous Execution:** The primary automation engine (`BS.exe`), physical hardware manager (`devmngd.exe`), co-processor FPGA logic, and UI server (`rui_svrd.exe`) execute entirely out of local flash storage (`/remote/Store`) and system memory.
2. **Local Staging & Activation:** Programming software interacts strictly through local LAN protocols:
   - Controller discovery via UDP broadcast (port 61510) and mDNS (port 5353).
   - Project archive transfer via anonymous/authenticated FTP (port 21) into `/remote/Store/updateroot`.
   - Project activation and engine reload via local framed TCP command packets to port 51510.
3. **Local Clock Preservation:** Although the controller attempts to query `www.urcrms.com/service_api/tc/get_utc` for time updates, failure to reach this endpoint does not prevent operation; the system maintains clock accuracy using its onboard hardware real-time clock (RTC) battery and standard `hwclock` utilities.
4. **Absence of Autonomous Firmware Polling:** The controller contains no background cron jobs, autonomous update daemons, or scheduled polling tasks that contact vendor cloud servers to check for or download firmware updates. While `rmsmcsman.exe` possesses an on-demand cloud file-transfer engine (`http://filesvr1.urcmcs.com/api/FileManager/predownload`) that stages files into `/updateroot/direct_download`, this mechanism is strictly interactive and dealer-driven via the remote RMS portal.
5. **WAN-Severed Subsystem Boundaries:** When WAN connectivity is completely severed (or when deployed on an air-gapped VLAN), only the following cloud-dependent ancillary services fail:
   - Off-site dealer remote maintenance and reverse signaling tunnels (RMS via SIP port 5061).
   - Live weather forecasts on touchscreen and remote widgets (`weatherservice.urcmx.com`).
   - Commercial third-party driver cloud license validation (`apikit.urcmcs.com`).
   - Off-LAN mobile push notifications (`gcm-http.googleapis.com`, `fcm.googleapis.com`).
   - System alert emails delivered via the default URC cloud relay (`mail.urcrms.com`).

   All local macro processing, sensor triggers, relay/IR/serial hardware control, two-way driver execution, local mobile/touchscreen UI rendering, and local programming updates remain 100% operational.
6. **Network Isolation Requirements:** Because `rmsmcsman.exe` contains the Azure RMS server IP (`20.121.207.6:5061`) as a hardcoded literal, DNS-based domain sinkholing alone is insufficient to prevent outbound connections. True vendor isolation requires default-deny WAN egress filtering at the network firewall or assigning the controller to a subnet with no default gateway. Furthermore, because local staging ports (FTP port 21, query port 51520, and reload trigger ports 51510/61520) lack authentication, LAN access to the controller should be restricted to authorized programmer hosts.

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
- **Action:** Transmit UDP discovery probe frame to `255.255.255.255:61510`.
- **Expected Response:** Controller transmits unicast UDP reply containing controller IP, MAC address, and model identifier string `[CONFIRMED]`.

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
   - *Resolution Plan:* Extract and hex-dump multiple dynamic command files during active touchscreen pairing sessions.
2. **DeviceGuard Local REST Endpoint Schemas (Port 54211):**
   - *Status:* Open.
   - *Problem Statement:* Document the complete JSON schema for `/deviceguard_api/settcroominfo` and `/deviceguard_api/updatebsinfo`.
   - *Resolution Plan:* Trace local loopback HTTP payloads between `BS.exe` and `deviceguard` using loopback packet capture.
3. **On-Demand Socket Activation Handshake (Port 22222):**
   - *Status:* Open.
   - *Problem Statement:* Determine the expected client stimulus frame required to interact with `/remote/App/svc_exam` when triggered on TCP port 22222.
   - *Resolution Plan:* Connect to port 22222 on a test controller, observe process spawning in `ps`, and capture protocol response.
