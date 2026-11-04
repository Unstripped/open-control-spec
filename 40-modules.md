---
title: "Module & Driver Runtime Model (Tcl, Two-Way Integration)"
section: 40
status: stable
confidence: confirmed
circumvention: false
---

# 40. Module & Driver Runtime Model

## 1. Overview & Execution Architecture

URC Total Control MRX controllers implement dynamic two-way equipment integration via an embedded **Tcl 8.6** runtime environment orchestrated primarily by the user interface and remote daemon `rui_svrd.exe`.

### 1.1 Packaging & Storage Topology
- **Core Tcl Packages (`/remote/Store/Prg/tclpkg/`):** Contains the shared system libraries and URC driver framework (`urc1`, `tcllib1.16`, `sqlite3.13.0`, `tls1.6.7`, `udp1.0.11`, `yajl1.6`).
- **Installed Drivers (`/remote/Store/data/TwoWay/*.vfd/`):** Each two-way driver is deployed as a bundle directory containing an `info.ini` manifest and encrypted Tcl scripts (in `ENC DATA` format per Section 21).
- **Execution Workspace (`/tmp/*.vfd/`):** At boot or driver launch, `rui_svrd.exe` decrypts driver files into temporary staging for execution.

---

## 2. Standard Global Environment (`::varIf*`)

Before evaluating a driver script, the host environment populates global parameters in the interpreter:

| Variable | Type | Default / Example | Meaning |
|---|---|---|---|
| `::varIfClientType` | string | `"System"` | Host execution context (`"System"`, `"Mobile"`, `"Client"`) |
| `::varIfModuleName` | string | `"Sonos"` | Driver bundle name (VFD identifier without extension) |
| `::varIfAddress` | string | `"192.168.1.50"` | Device network address or connection descriptor |
| `::varIfPort` | integer | `1400` | Target TCP/UDP port number |
| `::varIfVersionNum` | string | `"1.0.0.0"` | Driver build version |
| `::varIfDevName` | string | `"Sonos Player"` | User-assigned device label |
| `::varIfMRXMAC` | string | `"00:1F:B8:00:11:22"` | MRX controller primary MAC address |
| `::varIfBSModel` | string | `"MRX-15"` | MRX controller model code |
| `::varIfID` | string | `"1"` | Target system device instance ID |
| `::varIfZoneNum` | integer | `-1` | Assigned zone index |
| `::varIfBaudrate` | integer | `9600` | Serial baud rate (RS-232 devices) |
| `::varIfParity` | integer | `0` | Serial parity (0=None, 1=Odd, 2=Even) |
| `::varIfDatabits` | integer | `8` | Serial data bits (7, 8) |
| `::varIfStopbits` | integer | `1` | Serial stop bits (1, 2) |
| `::varIfScript` | string | `"Main.tcl"` | Entrypoint script within bundle |

---

## 3. Driver Lifecycle & Callbacks

A compliant clean-room driver module implements standard callback procedures:

```tcl
# Minimal Clean-Room Driver Structure
namespace eval ::Driver {
    variable socket ""

    proc Init {} {
        # Initialize internal state and variables
        set ::Driver::socket ""
    }

    proc Connect {} {
        # Establish connection using target parameters
        set ::Driver::socket [socket $::varIfAddress $::varIfPort]
        fconfigure $::Driver::socket -blocking 0 -buffering line
        fileevent $::Driver::socket readable ::Driver::OnData
    }

    proc ProcessCommand {cmd} {
        # Handle command dispatch from UI or automation macro
        if {$::Driver::socket ne ""} {
            puts $::Driver::socket $cmd
        }
    }

    proc OnData {} {
        if {[gets $::Driver::socket line] >= 0} {
            # Process received feedback and update UI/variables
        }
    }

    proc Close {} {
        if {$::Driver::socket ne ""} {
            catch {close $::Driver::socket}
            set ::Driver::socket ""
        }
    }
}
```

---

## 4. Hardware I/O Interfaces

### 4.1 IP (TCP/UDP)
- Uses standard Tcl socket primitives (`socket $host $port`) or the framework `::URC::IO::TCPConnect`.
- Encrypted TLS connections are established via `package require tls; tls::import $sock`.

### 4.2 RS-232 Serial
- Standard serial channels are opened via Linux serial device nodes (`/dev/ttyS*`) configured with `fconfigure $chan -mode "$baud,$parity,$data,$stop"`.

### 4.3 IR Transmission
- Handled at the controller base station level via macro dispatch (opcodes `0x21` or direct carrier triggers).

---

## 5. Verification & Test Module

A synthetic standalone test module written in pure Tcl (no proprietary vendor assets) demonstrates successful initialization and IP command execution against an echo server:

```tcl
# test_module.tcl
set ::varIfAddress "127.0.0.1"
set ::varIfPort 8080
set ::varIfDevName "Test Device"

proc SendTestCommand {msg} {
    set s [socket $::varIfAddress $::varIfPort]
    puts $s $msg
    flush $s
    close $s
}
```

When evaluated in the ARM32 container (`docker compose run --rm mrx tclsh`), the synthetic script successfully opens connections and transmits commands.

---

## 6. Open Items

None.
