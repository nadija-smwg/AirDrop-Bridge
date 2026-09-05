# UniversalDrop — Phase-Wise Implementation Plan

**Date:** September 2026
**Status:** Research-backed plan ready for execution

---

## Phase Overview

```
PHASE 0 ── Research & Feasibility ──────────────── ✅ COMPLETE (this document)
    │
    ▼
PHASE 1 ── Development Lab & Protocol Lab ──────── Entry point
    │
    ▼
PHASE 2 ── Linux AirDrop Receiver (AWDL+AirDrop)─ Core proof
    │
    ▼
PHASE 3 ── iPhone → Linux Real Transfer ─────────  MVP 1
    │
    ▼
PHASE 4 ── Native UniversalDrop Protocol ─────────  Core protocol
    │
    ▼
PHASE 5 ── Android Native App ───────────────────── MVP 2 (partial)
    │
    ▼
PHASE 6 ── Windows Native App ──────────────────── MVP 2 (complete)
    │
    ▼
PHASE 7 ── Android AirDrop Receiver ─────────────── MVP 3 (if feasible)
    │
    ▼
PHASE 8 ── Windows AirDrop Bridge ───────────────── MVP 4 (if feasible)
    │
    ▼
PHASE 9 ── Security Hardening ──────────────────── Production readiness
    │
    ▼
PHASE 10 ── Cross-Platform Integration ──────────── Full integration
    │
    ▼
PHASE 11 ── Compatibility Testing ───────────────── Validation
    │
    ▼
PHASE 12 ── Production UX & Release ─────────────── MVP 5
```

---

## PHASE 0 — RESEARCH & FEASIBILITY

**Objective:** Produce a comprehensive engineering report based on deep technical investigation.

**Status:** ✅ COMPLETE

**Deliverables:**
- [x] Executive summary
- [x] Feasibility report
- [x] AirDrop protocol analysis
- [x] AWDL research
- [x] BLE discovery analysis
- [x] Platform feasibility (Linux, Windows, Android)
- [x] Hardware compatibility matrix
- [x] Native protocol design
- [x] Security threat model
- [x] Risk register
- [x] Phase-wise plan

**Gate:** ✅ GO — Sufficient evidence exists to proceed to Phase 1.

---

## PHASE 1 — DEVELOPMENT LAB & PROTOCOL LAB

### Objective
Set up the development environment, establish the repository structure, build the packet analysis lab, and create the foundational Rust workspace.

### Why This Phase Exists
Before writing protocol code, we need:
1. A reproducible development environment
2. Verified hardware (Wi-Fi adapter, Bluetooth, test devices)
3. The ability to capture and analyze AirDrop traffic
4. A clean repository structure with CI

### Dependencies
- Phase 0 complete ✅

### Prerequisites
```
Hardware:
  - Linux PC (bare metal, NOT VM)
  - Alfa AWUS036ACH (RTL8812AU) OR TP-Link TL-WN722N v1 (AR9271)
  - USB Bluetooth 4.0+ adapter (if no internal BT)
  - iPhone (iOS 17+)
  - USB-C/Lightning cable (for initial connectivity testing)

Software:
  - Ubuntu 24.04 LTS or Kali Linux
  - Kernel 6.8+
  - Wireshark 4.x
  - BlueZ 5.x
  - Rust 1.79+ (stable)
  - Git
```

### Technologies
- Rust, Cargo workspaces
- Wireshark (packet analysis)
- tcpdump (network capture)
- hcitool / bluetoothctl (BLE)
- airmon-ng (Wi-Fi monitor mode testing)

### Architecture

```
universaldrop/
├── Cargo.toml                  # Workspace root
├── crates/
│   ├── ud-core/                # Shared types, traits, utilities
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── ud-crypto/              # Crypto primitives, key management
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── ud-protocol/            # Native UniversalDrop protocol messages
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── ud-transfer/            # File chunking, resume, integrity
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── ud-airdrop/             # AirDrop protocol implementation
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── ud-awdl/                # AWDL link layer
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── ud-ble/                 # BLE advertising/scanning
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   └── ud-mdns/                # mDNS service discovery
│       ├── Cargo.toml
│       └── src/lib.rs
├── apps/
│   └── linux-receiver/         # Linux CLI receiver
│       ├── Cargo.toml
│       └── src/main.rs
├── tools/
│   ├── ble-scanner/            # BLE analysis tool
│   ├── awdl-monitor/           # AWDL traffic monitor
│   └── packet-analyzer/        # Protocol packet analyzer
├── docs/                       # This DOC folder
├── tests/
│   ├── unit/
│   └── integration/
├── .github/
│   └── workflows/
│       └── ci.yml
├── LICENSE-APACHE
├── LICENSE-MIT
├── THIRD_PARTY_NOTICES
└── README.md
```

### Implementation Tasks

1. **Hardware verification**
   - [ ] Verify Wi-Fi adapter enters monitor mode: `sudo airmon-ng start wlan1`
   - [ ] Verify packet injection: `sudo aireplay-ng --test wlan1mon`
   - [ ] Verify Bluetooth adapter works: `hciconfig hci0 up; hcitool lescan`
   - [ ] Verify iPhone is discoverable: initiate AirDrop share on iPhone, capture BLE

2. **Packet capture lab**
   - [ ] Capture AirDrop BLE advertisements from iPhone using `hcidump`
   - [ ] Capture AWDL frames using Wireshark on monitor interface
   - [ ] Capture mDNS traffic during AirDrop discovery
   - [ ] Document packet formats with annotated captures
   - [ ] Save reference captures to `docs/captures/`

3. **Repository setup**
   - [ ] Initialize Cargo workspace
   - [ ] Create crate stubs with proper dependencies
   - [ ] Set up CI (GitHub Actions: build, test, clippy, fmt)
   - [ ] Create LICENSE-APACHE and LICENSE-MIT files
   - [ ] Create THIRD_PARTY_NOTICES

4. **Foundation code**
   - [ ] `ud-core`: Error types, logging framework (tracing), config
   - [ ] `ud-crypto`: Key generation (Ed25519), self-signed cert generation
   - [ ] BLE scanner tool: scan and decode Apple BLE advertisements

### Tests
- Wi-Fi adapter enters monitor mode
- Packet injection test passes
- BLE scan detects iPhone AirDrop advertisements
- Cargo workspace builds and all stubs compile
- CI pipeline passes

### Success Criteria
```
✅ Wi-Fi adapter confirmed in monitor mode with injection
✅ BLE scanner detects iPhone AirDrop advertisement
✅ At least one AWDL frame captured in Wireshark
✅ Cargo workspace compiles with all crate stubs
✅ CI pipeline runs successfully
✅ Packet captures documented
```

### Failure Criteria
```
🔴 Wi-Fi adapter does not support monitor mode → acquire different hardware
🔴 No AWDL frames visible → check channel, verify iPhone is sharing
🔴 BLE advertisements not detected → check Bluetooth hardware/drivers
```

### Risks
| Risk | Mitigation |
|------|------------|
| Wi-Fi adapter doesn't support injection | Have backup adapter (AR9271) |
| Linux kernel version incompatible | Use Kali Linux (pre-configured) |
| iPhone not generating AWDL traffic | Ensure iPhone is actively sharing |

### Fallback
If the Alfa adapter fails, use TP-Link TL-WN722N v1 (AR9271). If both fail, purchase a confirmed-compatible adapter from the OWL project's tested hardware list.

### Deliverables
- [ ] Working development environment
- [ ] Verified hardware setup
- [ ] Packet capture documentation
- [ ] Cargo workspace with crate stubs
- [ ] CI pipeline
- [ ] BLE scanner tool

### Evidence
- OWL documentation confirms AR9271 and RTL8812AU compatibility
- opendrop-rs README describes hardware requirements
- Linux kernel 6.14+ mainlines RTL8812AU driver

### Estimated Complexity: **Medium**

### Gate: **CONDITIONAL GO**
Condition: Hardware must be verified working. If hardware verification fails, Phase 1 is BLOCKED until compatible hardware is acquired.

---

## PHASE 2 — LINUX AIRDROP RECEIVER

### Objective
Build a working AWDL + AirDrop receiver on Linux that can be discovered by an iPhone and accept a /Discover request.

### Why This Phase Exists
This is the core technical proof. If we cannot implement AWDL and respond to /Discover on Linux with verified hardware, the entire AirDrop compatibility path is blocked.

### Dependencies
- Phase 1 complete (hardware verified, workspace set up)

### Prerequisites
- All Phase 1 hardware
- Captured reference packets from Phase 1

### Technologies
- Rust
- `libpcap` (via `pcap` crate) for frame injection/capture
- `netlink` for network interface management
- `rustls` for TLS
- Binary plist parsing (`plist` crate)
- Custom AWDL frame parser/generator

### Architecture

```
┌────────────────────────────────────────────┐
│              linux-receiver CLI             │
│                                            │
│  ┌──────────┐ ┌──────────┐ ┌────────────┐ │
│  │ ud-ble   │ │ ud-awdl  │ │ ud-airdrop │ │
│  │ BLE adv  │ │ AWDL     │ │ HTTP/TLS   │ │
│  │ scanning │ │ link     │ │ /Discover  │ │
│  │          │ │ layer    │ │ /Ask       │ │
│  │          │ │          │ │ /Upload    │ │
│  └────┬─────┘ └────┬─────┘ └─────┬──────┘ │
│       │            │              │        │
│  ┌────┴────────────┴──────────────┴──────┐ │
│  │           ud-core / ud-crypto          │ │
│  │    Logging, config, key management     │ │
│  └────────────────────────────────────────┘ │
└────────────────────────────────────────────┘
         │              │             │
    Bluetooth       Wi-Fi (mon)    Wi-Fi (mon)
    HCI/BlueZ      AWDL frames    TCP/HTTP
```

### Implementation Tasks

1. **AWDL Link Layer (`ud-awdl`)**
   - [ ] Parse AWDL action frames (MIF, PSF, election, data)
   - [ ] Implement AWDL frame generation
   - [ ] Implement channel management (social channels 6, 44, 149)
   - [ ] Implement availability window timing
   - [ ] Implement master election participation
   - [ ] Implement synchronization to existing AWDL master
   - [ ] Create virtual network interface for AWDL traffic
   - [ ] Implement unicast and multicast frame handling

2. **mDNS Service (`ud-mdns`)**
   - [ ] Advertise `_airdrop._tcp` service on AWDL interface
   - [ ] Handle mDNS queries and responses
   - [ ] Manage service registration/deregistration

3. **AirDrop Protocol (`ud-airdrop`)**
   - [ ] Implement HTTPS server on AWDL interface
   - [ ] Handle POST /Discover request
   - [ ] Parse binary plist request body
   - [ ] Generate binary plist response
   - [ ] Generate self-signed TLS certificate
   - [ ] Implement /Discover response with receiver info

4. **BLE Advertising (`ud-ble`)**
   - [ ] Generate Apple-format BLE advertisement (Company ID 0x004C, type 0x05)
   - [ ] Broadcast advertisement via BlueZ HCI
   - [ ] Implement rotating advertisement

5. **Integration**
   - [ ] Wire all components into linux-receiver CLI
   - [ ] Structured logging with categories: [BLE] [AWDL] [MDNS] [AIRDROP] [TLS]
   - [ ] Configuration file support

### Tests
- Unit tests for AWDL frame parsing/generation
- Unit tests for binary plist parsing
- Unit tests for mDNS query/response
- Integration test: start receiver, verify mDNS advertisement visible
- Integration test: verify AWDL sync to iPhone's AWDL master

### Success Criteria
```
✅ AWDL link layer syncs to iPhone's AWDL cluster
✅ mDNS _airdrop._tcp service visible from iPhone
✅ iPhone shows UniversalDrop in AirDrop device list
✅ /Discover request received and responded to successfully
✅ All structured logs show complete protocol flow
```

### Failure Criteria
```
🔴 Cannot sync to AWDL cluster → investigate frame timing, channel selection
🔴 iPhone doesn't discover receiver → check BLE advertisement format, mDNS
🔴 TLS handshake fails → check certificate generation, cipher suites
```

### Risks
| Risk | Mitigation |
|------|------------|
| AWDL sync timing is wrong | Study opendrop-rs timing; iterative testing |
| Frame injection unreliable | Test with multiple adapters; adjust injection rate |
| mDNS not visible over AWDL | Verify AWDL interface IP config; check multicast |

### Fallback
If independent AWDL implementation fails, consider running OWL (GPLv3) as a separate process for the AWDL layer only, communicating via IPC. This isolates the GPLv3 code. This is a **temporary research fallback**, not a production strategy.

### Deliverables
- [ ] Working `ud-awdl` crate
- [ ] Working `ud-mdns` crate (AirDrop-specific)
- [ ] Working `ud-airdrop` crate (partial: /Discover)
- [ ] Working `ud-ble` crate (advertising)
- [ ] linux-receiver shows up in iPhone's AirDrop list

### Evidence
- opendrop-rs demonstrates this is achievable on iOS 18
- OWL demonstrates AWDL sync is possible from userspace
- SEEMOO Lab published detailed AWDL frame specifications

### Estimated Complexity: **Very High**

### Gate: **CONDITIONAL GO**
Condition: iPhone must discover the linux-receiver and /Discover must succeed. If AWDL sync fails completely, evaluate whether the approach is viable with available hardware.

---

## PHASE 3 — IPHONE → LINUX REAL TRANSFER (MVP 1)

### Objective
Complete the full transfer flow: iPhone sends a file via AirDrop, Linux receiver accepts and saves it.

### Why This Phase Exists
This is **MVP 1** — the minimum viable proof that AirDrop interoperability works.

### Dependencies
- Phase 2 complete (iPhone discovers receiver)

### Prerequisites
- Phase 2 deliverables
- iPhone with test files (photos, documents)

### Technologies
- Everything from Phase 2
- `ud-transfer` crate for file handling
- CPIO archive parsing
- DVZip decompression (if used)

### Architecture

```
iPhone                     linux-receiver
  │                              │
  │── POST /Discover ──────────► │ ← Already working (Phase 2)
  │◄─── Response ────────────────│
  │                              │
  │── POST /Ask ───────────────► │ ← NEW: Parse file metadata
  │   {FileName, FileType, ...}  │    Show accept/decline prompt
  │◄─── Accept ─────────────────│    in terminal
  │                              │
  │── POST /Upload ────────────► │ ← NEW: Receive CPIO archive
  │   {CPIO archive data}        │    Extract files
  │◄─── Success ────────────────│    Save to disk
  │                              │
  │   Transfer complete ✅       │
```

### Implementation Tasks

1. **AirDrop /Ask handler**
   - [ ] Parse /Ask binary plist (file metadata)
   - [ ] Extract file names, sizes, types
   - [ ] Display transfer request in terminal
   - [ ] Accept/decline user prompt
   - [ ] Generate accept/reject response

2. **AirDrop /Upload handler**
   - [ ] Receive upload data stream
   - [ ] Parse CPIO archive format
   - [ ] Handle DVZip decompression
   - [ ] Extract files from archive
   - [ ] Sanitize file names (path traversal prevention)
   - [ ] Write files to temporary location
   - [ ] Atomic move to final destination
   - [ ] Generate success response

3. **File handling (`ud-transfer`)**
   - [ ] CPIO archive parser
   - [ ] DVZip decompressor
   - [ ] File name sanitizer
   - [ ] Secure temporary file management
   - [ ] Progress tracking
   - [ ] Integrity verification

4. **Testing with real iPhone**
   - [ ] Test with single photo (HEIC)
   - [ ] Test with single photo (JPEG)
   - [ ] Test with video
   - [ ] Test with document (PDF)
   - [ ] Test with multiple files
   - [ ] Test with large file (100 MB+)
   - [ ] Test cancel mid-transfer
   - [ ] Test with different iPhone models
   - [ ] Test with different iOS versions

### Tests
- Unit tests for CPIO parsing
- Unit tests for file name sanitization
- Unit tests for /Ask plist parsing
- Integration test: full transfer flow
- Real-device test matrix

### Success Criteria
```
✅ iPhone sends photo via AirDrop to linux-receiver
✅ File is received intact (hash matches)
✅ Multiple file types work (HEIC, JPEG, PDF, video)
✅ Transfer of 100 MB+ file completes
✅ Cancel mid-transfer works cleanly
✅ No path traversal vulnerability
✅ Files saved with correct names
```

### Failure Criteria
```
🔴 /Ask request fails → check plist format, TLS session continuity
🔴 /Upload data corrupted → check CPIO parsing, stream handling
🔴 iPhone rejects after /Ask → check response format
🔴 Large files stall → investigate half-duplex limitations, chunk/ACK timing
```

### Risks
| Risk | Mitigation |
|------|------------|
| TLS session must be same for Ask/Upload | Implement session persistence |
| Large files stall on half-duplex radio | Implement flow control; test with smaller files first |
| CPIO format variations across iOS versions | Test with multiple iOS versions |
| HEIC files need special handling | Save raw; conversion is not Phase 3 scope |

### Fallback
If large files consistently fail, implement a maximum file size limit for AirDrop transfers and document the limitation. Focus on making small/medium transfers reliable.

### Deliverables
- [ ] Complete AirDrop receiver (Discover + Ask + Upload)
- [ ] CPIO parser
- [ ] File sanitization module
- [ ] Real-device test results documented
- [ ] **MVP 1 ACHIEVED: iPhone → Linux file transfer works**

### Evidence
- opendrop-rs has validated this exact flow on iOS 18
- CPIO and binary plist formats are well-documented

### Estimated Complexity: **High**

### Gate: **GO** (if Phase 2 passes)
MVP 1 must be fully working before proceeding to Phase 4. If it works, we have confirmed that the core AirDrop interoperability goal is achievable.

---

## PHASE 4 — NATIVE UNIVERSALDROP PROTOCOL

### Objective
Design and implement the native UniversalDrop protocol for direct Android ↔ Windows transfers, independent of AirDrop.

### Why This Phase Exists
The native protocol serves two purposes:
1. It provides file transfer between platforms where AirDrop is not feasible (most Android/Windows configurations)
2. It is the fallback transfer mechanism when AirDrop hardware is unavailable

### Dependencies
- Phase 1 complete (workspace, crypto)
- Phase 3 NOT required (native protocol is independent)

### Technical Question Being Answered
Can we build a high-performance, resumable, encrypted file transfer protocol that works reliably across Android and Windows over local Wi-Fi?

### Technologies
- Rust
- `tokio` (async runtime)
- `rustls` (TLS 1.3)
- `serde` (serialization)
- `mdns-sd` (service discovery)
- `btleplug` (BLE — optional for proximity)
- `ring` / `sha2` (integrity)
- `ed25519-dalek` (device identity)

### Architecture

```
┌─────────────────────────────────────────┐
│           UniversalDrop Native          │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │ Discovery │  │ Protocol │  │  File │ │
│  │ mDNS/BLE │  │ Messages │  │ Xfer  │ │
│  └─────┬────┘  └─────┬────┘  └───┬───┘ │
│        │             │            │     │
│  ┌─────┴─────────────┴────────────┴───┐ │
│  │        TCP + TLS 1.3 Transport      │ │
│  └─────────────────────────────────────┘ │
│                                         │
│  ┌─────────────────────────────────────┐ │
│  │   Security: Keys, Certs, Pairing    │ │
│  └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### Implementation Tasks

1. **Protocol messages (`ud-protocol`)**
   - [ ] Define message header format (magic, version, type, flags, length)
   - [ ] Implement all message types (HELLO through ERROR)
   - [ ] Binary serialization/deserialization
   - [ ] Message validation
   - [ ] Protocol version negotiation

2. **Transport layer**
   - [ ] TCP listener with TLS 1.3 (rustls)
   - [ ] Connection management
   - [ ] Session handling
   - [ ] Timeout management
   - [ ] Connection pooling (for multiple transfers)

3. **Discovery**
   - [ ] mDNS service advertisement (`_universaldrop._tcp`)
   - [ ] mDNS service discovery
   - [ ] Optional BLE proximity detection
   - [ ] Device info exchange

4. **File transfer (`ud-transfer` expansion)**
   - [ ] Chunked streaming (1 MB chunks, configurable)
   - [ ] Per-chunk acknowledgment
   - [ ] Per-file SHA-256 integrity verification
   - [ ] Resume protocol (offset-based)
   - [ ] Multi-file transfer
   - [ ] Directory transfer (recursive)
   - [ ] Progress reporting
   - [ ] Cancellation
   - [ ] Rate limiting

5. **Security (`ud-crypto` expansion)**
   - [ ] Ed25519 device identity keypair generation
   - [ ] Self-signed TLS certificate from device keypair
   - [ ] Trust-On-First-Use (TOFU) certificate pinning
   - [ ] Optional pairing mechanism
   - [ ] Secure key storage (OS keychain integration)

6. **Linux CLI integration**
   - [ ] `universaldrop send <files>` command
   - [ ] `universaldrop receive` command
   - [ ] `universaldrop discover` command
   - [ ] Progress bars, structured logging

### Tests
- Unit tests: message serialization/deserialization round-trips
- Unit tests: file chunking, SHA-256 verification
- Unit tests: protocol state machine
- Integration: Linux ↔ Linux transfer
- Integration: 100 MB file transfer
- Integration: Resume after disconnect
- Integration: Multiple file transfer
- Integration: Cancel mid-transfer

### Required Hardware
- Two Linux machines (or one machine with loopback for testing)
- Standard Wi-Fi (no special hardware needed)

### Success Criteria
```
✅ Two Linux machines discover each other via mDNS
✅ File transfer completes with integrity verification
✅ Resume works after simulated disconnect
✅ 1 GB file transfers successfully
✅ Multiple files transfer in single session
✅ TLS encryption verified in Wireshark
✅ Protocol messages are well-documented
```

### Failure Criteria
```
🔴 mDNS discovery fails across subnets → expected; document LAN-only limitation
🔴 Resume mechanism doesn't work → debug offset tracking
🔴 Performance < 10 MB/s on LAN → optimize buffer sizes, chunk handling
```

### Risks
| Risk | Mitigation |
|------|------------|
| mDNS doesn't work across VLANs | Document LAN-only requirement |
| TLS handshake adds latency | Optimize; consider 0-RTT for paired devices |
| Protocol design needs revision | Version field allows future changes |

### Fallback
If TCP+TLS performance is insufficient, evaluate QUIC (quinn crate). The protocol message layer is transport-agnostic by design.

### Deliverables
- [ ] Complete `ud-protocol` crate
- [ ] Complete `ud-transfer` crate (with resume)
- [ ] TCP+TLS transport implementation
- [ ] mDNS discovery
- [ ] Linux CLI sender/receiver
- [ ] Protocol specification document

### Estimated Complexity: **High**

### Gate: **GO**
This phase can proceed in parallel with Phase 2/3 if resources allow, since it is independent of the AirDrop stack.

---

## PHASE 5 — ANDROID NATIVE APP

### Objective
Build the Android app with native UniversalDrop protocol support (Android ↔ Android, Android ↔ Windows/Linux).

### Why This Phase Exists
Android is the most important target platform for the native protocol. Many users want to share files between Android and Windows without cloud services.

### Dependencies
- Phase 4 complete (native protocol working on Linux)

### Technologies
- Kotlin (Android UI)
- Rust core via JNI/NDK (`android-ndk-rs`)
- Jetpack Compose (Material 3 UI)
- Android `NsdManager` (mDNS)
- Android `BluetoothLeAdvertiser` (BLE)
- Foreground Service (background operation)

### Architecture

```
┌──────────────────────────────────────┐
│         Android Application          │
│                                      │
│  ┌────────────────────────────────┐  │
│  │   Kotlin UI (Jetpack Compose)  │  │
│  │   Material 3 / Dynamic Color   │  │
│  └────────────┬───────────────────┘  │
│               │ JNI                  │
│  ┌────────────┴───────────────────┐  │
│  │   Rust Core (via NDK)          │  │
│  │   ud-protocol, ud-transfer,    │  │
│  │   ud-crypto, ud-mdns           │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │   Android Platform Layer       │  │
│  │   NsdManager, BLE, Storage,    │  │
│  │   Notifications, Permissions   │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### Implementation Tasks

1. **Rust-Android bridge**
   - [ ] Set up cross-compilation for Android (aarch64, armv7, x86_64)
   - [ ] JNI bindings for core Rust functions
   - [ ] Async bridge (Kotlin coroutines ↔ Rust tokio)

2. **Android UI**
   - [ ] Main screen: discoverable toggle, nearby devices list
   - [ ] Send screen: file picker, device selection
   - [ ] Receive screen: incoming request dialog (accept/decline)
   - [ ] Transfer progress screen
   - [ ] History screen
   - [ ] Settings screen (device name, save location, visibility)

3. **Android platform integration**
   - [ ] Foreground service for background receiving
   - [ ] Notification for incoming requests
   - [ ] Scoped storage file access
   - [ ] Wi-Fi/Bluetooth permission handling (Android 12+ model)
   - [ ] Battery optimization exemption request
   - [ ] Share intent receiver (share from other apps)

4. **Testing**
   - [ ] Android ↔ Linux transfer
   - [ ] Android ↔ Android transfer
   - [ ] Background receive
   - [ ] Large file transfer
   - [ ] Multiple Android versions (12, 13, 14, 15)

### Success Criteria
```
✅ Android app discovers Linux/other Android devices
✅ File transfer Android → Linux works
✅ File transfer Linux → Android works
✅ Background receive works via foreground service
✅ Notifications show incoming requests
✅ 100 MB file transfer succeeds
✅ Resume works after app backgrounding
```

### Estimated Complexity: **High**

### Gate: **GO** (conditional on Phase 4)

---

## PHASE 6 — WINDOWS NATIVE APP

### Objective
Build the Windows app with native UniversalDrop protocol support.

### Dependencies
- Phase 4 complete (native protocol)

### Technologies
- Rust core
- WinRT APIs (BLE, mDNS)
- UI: Rust + egui OR C#/WinUI 3 frontend with Rust backend
- Windows Service (background operation)

### Implementation Tasks

1. **Windows platform layer**
   - [ ] WinRT BLE advertising and scanning
   - [ ] Windows mDNS integration
   - [ ] Windows credential store for keys
   - [ ] Windows notification API
   - [ ] Windows Service for background operation

2. **Windows UI**
   - [ ] System tray icon
   - [ ] Main window (discover, send, receive)
   - [ ] Notification toast for incoming requests
   - [ ] File explorer integration (context menu "Send via UniversalDrop")

3. **Testing**
   - [ ] Windows ↔ Linux transfer
   - [ ] Windows ↔ Android transfer
   - [ ] Windows ↔ Windows transfer
   - [ ] Background receive as Windows Service

### Success Criteria
```
✅ Windows app discovers devices on LAN
✅ File transfer Windows ↔ Android works
✅ Background receive via Windows Service
✅ System tray integration works
✅ 1 GB file transfer succeeds
```

### Estimated Complexity: **High**

### Gate: **GO** (conditional on Phase 4)
**MVP 2 is achieved when Phase 5 + Phase 6 are both complete.**

---

## PHASE 7 — ANDROID AIRDROP RECEIVER (MVP 3)

### Objective
Make an Android device appear as an AirDrop-compatible receiver to an iPhone.

### Why This Phase Exists
This is the most ambitious goal — making iPhones send files to Android devices via AirDrop without any iOS app.

### Dependencies
- Phase 3 complete (Linux AirDrop receiver proven)
- Phase 5 complete (Android app exists)

### Technical Question Being Answered
Can a non-rooted Android device perform AWDL or equivalent P2P communication to receive AirDrop transfers?

### Technologies
- Rust core (AirDrop protocol from Phase 2/3)
- Android Wi-Fi Aware (NAN) API — if Apple adopts Wi-Fi Aware (DMA)
- Android BLE advertising
- JNI bridge

### Strategy Options

| Option | Feasibility | When |
|--------|-------------|------|
| **A: Wi-Fi Aware (DMA-compliant)** | 🟡 PROBABLE | When Apple ships Wi-Fi Aware AirDrop (2026-2027?) |
| **B: Root + custom driver** | 🧪 EXPERIMENTAL | Now, but tiny audience |
| **C: Relay via Linux bridge** | 🟡 PROBABLE | Now, requires intermediate Linux device |

### Implementation Tasks

**If Wi-Fi Aware path (Option A) becomes available:**
1. [ ] Implement Wi-Fi Aware peer-to-peer connection
2. [ ] Port AirDrop protocol layer to run over Wi-Fi Aware
3. [ ] Port BLE advertising to Android
4. [ ] Integrate with Android UI from Phase 5

**If relay path (Option C):**
1. [ ] Linux bridge receives AirDrop, forwards via native protocol to Android
2. [ ] Transparent to user (bridge auto-discovers Android device)

### Success Criteria
```
✅ iPhone discovers Android device in AirDrop
✅ File transfers from iPhone to Android via AirDrop
✅ No iOS app required on iPhone
```

### Estimated Complexity: **Research-grade**

### Gate: **CONDITIONAL GO**
Blocked until either (a) Apple ships Wi-Fi Aware AirDrop support, or (b) a viable non-root AWDL mechanism is discovered. In the meantime, maintain the relay bridge (Option C) as a demonstration.

---

## PHASE 8 — WINDOWS AIRDROP BRIDGE (MVP 4)

### Objective
Enable Windows to receive AirDrop transfers from iPhones.

### Dependencies
- Phase 3 complete (Linux AirDrop receiver)
- Phase 6 complete (Windows app exists)

### Strategy Options

| Option | Feasibility | Complexity |
|--------|-------------|------------|
| **A: External USB adapter + Rust AWDL** | 🧪 EXPERIMENTAL | Very High |
| **B: Linux VM bridge** | 🟡 PROBABLE | High |
| **C: Wi-Fi Aware (DMA, future)** | 🟡 PROBABLE | Medium (when available) |

### Implementation Tasks

**Option B (most practical):**
1. [ ] Package Linux AirDrop receiver as lightweight VM/container
2. [ ] USB Wi-Fi adapter passthrough to VM
3. [ ] Bridge: VM receives AirDrop → forwards to Windows via native protocol
4. [ ] Windows UI shows transfer as "via AirDrop"

### Success Criteria
```
✅ iPhone AirDrop to Windows PC works (via bridge)
✅ User experience is reasonably seamless
✅ Setup is documented and reproducible
```

### Estimated Complexity: **Very High**

### Gate: **CONDITIONAL GO**
Depends on hardware and VM feasibility. This phase may be deferred if Wi-Fi Aware becomes available first.

---

## PHASE 9 — SECURITY HARDENING

### Objective
Comprehensive security review and hardening of all protocol implementations.

### Implementation Tasks
- [ ] Security audit of all crypto code
- [ ] Fuzz testing (protocol parsers, plist parser, CPIO parser)
- [ ] Penetration testing on exposed services
- [ ] Review all file handling for path traversal
- [ ] Review all deserialization for DoS vectors
- [ ] Secure key storage verification on all platforms
- [ ] TLS configuration hardening
- [ ] Rate limiting on all endpoints
- [ ] Resource exhaustion testing

### Estimated Complexity: **High**

### Gate: **GO** (after Phases 4-6)

---

## PHASE 10 — CROSS-PLATFORM INTEGRATION

### Objective
Ensure all platform combinations work together reliably.

### Test Matrix
```
            Linux    Android    Windows
Linux         ✅        ✅         ✅
Android       ✅        ✅         ✅
Windows       ✅        ✅         ✅
```

### Estimated Complexity: **Medium**

---

## PHASE 11 — COMPATIBILITY TESTING

### Objective
Test across diverse hardware and software configurations.

### Test Targets
- Multiple iPhone models (12, 13, 14, 15, 16)
- Multiple iOS versions (17, 18, 19)
- Multiple Android versions (12, 13, 14, 15)
- Multiple Android vendors (Samsung, Pixel, OnePlus, Xiaomi)
- Multiple Windows versions (10, 11)
- Multiple Wi-Fi chipsets

### Estimated Complexity: **Medium** (time-intensive)

---

## PHASE 12 — PRODUCTION UX & RELEASE (MVP 5)

### Objective
Polish the user experience and prepare for public release.

### Implementation Tasks
- [ ] Android: Material 3 polished UI
- [ ] Windows: Modern WinUI design
- [ ] App store listings (Google Play)
- [ ] Windows installer (MSIX/MSI)
- [ ] Documentation site
- [ ] Getting started guide
- [ ] FAQ and troubleshooting
- [ ] Contribution guide

### Estimated Complexity: **Medium**

---

## Phase Gates Summary

| Phase | Gate | Status |
|-------|------|--------|
| Phase 0 | Research complete | ✅ GO |
| Phase 1 | Hardware verified, workspace ready | ⏳ CONDITIONAL GO |
| Phase 2 | iPhone discovers receiver | ⏳ Depends on Phase 1 |
| Phase 3 | Full file transfer works | ⏳ Depends on Phase 2 |
| Phase 4 | Native protocol works (Linux ↔ Linux) | ⏳ Independent of Phase 2/3 |
| Phase 5 | Android app works | ⏳ Depends on Phase 4 |
| Phase 6 | Windows app works | ⏳ Depends on Phase 4 |
| Phase 7 | Android AirDrop receiver | ⏳ CONDITIONAL — awaiting DMA/Wi-Fi Aware |
| Phase 8 | Windows AirDrop bridge | ⏳ CONDITIONAL — depends on approach |
| Phase 9 | Security audit passes | ⏳ Depends on Phase 4-6 |
| Phase 10 | Cross-platform integration | ⏳ Depends on Phase 5-6 |
| Phase 11 | Compatibility testing | ⏳ Depends on Phase 10 |
| Phase 12 | Release ready | ⏳ Depends on Phase 11 |

### Parallel Execution Opportunities

```
                    Phase 0 (DONE)
                         │
                    Phase 1
                    /         \
              Phase 2/3      Phase 4
              (AirDrop)      (Native Protocol)
                   │          /        \
                   │      Phase 5    Phase 6
                   │      (Android)  (Windows)
                   │          \        /
                   │       Phase 9/10
                   │            │
              Phase 7/8    Phase 11
                   \          /
                    Phase 12
```

Phase 2/3 (AirDrop) and Phase 4 (Native Protocol) can be developed **in parallel**.
Phase 5 (Android) and Phase 6 (Windows) can be developed **in parallel**.
