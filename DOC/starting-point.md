# UniversalDrop — Recommended Starting Point & Immediate Next Steps

**Date:** September 2026

---

## 1. Recommended Starting Point

```
Start with:     PHASE 1 — Development Lab & Protocol Lab

Reason:         Hardware verification is the gate to everything else.
                No amount of code will work without confirmed-compatible
                Wi-Fi hardware. The lab setup also produces the packet
                captures that inform the AWDL and AirDrop implementations.

Hardware:       Linux PC (bare metal) + Alfa AWUS036ACH (RTL8812AU)
                or TP-Link TL-WN722N v1 (AR9271)
                USB Bluetooth 4.0+ adapter
                iPhone 12+ with iOS 17+

OS:             Ubuntu 24.04 LTS (primary) or Kali Linux (alternative)
                Kernel 6.8+

Language:       Rust (stable 1.79+)

First experiment:
                1. Put Wi-Fi adapter in monitor mode
                2. Open AirDrop share sheet on iPhone
                3. Capture AWDL action frames in Wireshark
                4. Capture BLE advertisement with hcitool/bluetoothctl
                5. Document captured frame formats

Expected result:
                AWDL action frames visible on social channel 6/44/149
                BLE advertisement with Apple Company ID 0x004C visible
                Wireshark shows decodable AWDL TLVs

If successful:  Proceed to PHASE 2 (AWDL link layer implementation)
                Highest-priority: sync to iPhone's AWDL cluster

If unsuccessful:
                1. Verify adapter chipset (not all versions have same chipset)
                2. Try alternative adapter
                3. Try Kali Linux (pre-configured wireless tools)
                4. Check if iPhone is generating AWDL (open share sheet)
                5. If all adapters fail: reassess hardware recommendations
```

---

## 2. Top 10 Technical Risks

| # | Risk | Severity | Status |
|---|------|----------|--------|
| 1 | AWDL requires special Wi-Fi hardware not available on most PCs | **CRITICAL** | ✅ CONFIRMED — must use external adapter |
| 2 | Windows cannot run AWDL natively | **CRITICAL** | ✅ CONFIRMED — needs bridge/VM |
| 3 | Android AWDL not accessible to third-party apps | **CRITICAL** | ✅ CONFIRMED — Google's implementation is private |
| 4 | Apple changes AirDrop/AWDL protocol in future iOS | **HIGH** | 🟡 PROBABLE — has happened before |
| 5 | Apple-signed identity required for "Contacts Only" | **HIGH** | ✅ CONFIRMED — "Everyone" mode only |
| 6 | Large file transfers stall on half-duplex monitor radio | **HIGH** | 🟡 PROBABLE — reported in opendrop-rs |
| 7 | BLE advertisement format rejected by newer iOS | **MEDIUM** | ❓ UNKNOWN — needs testing |
| 8 | GPLv3 code accidentally included in codebase | **MEDIUM** | Preventable — code review + CI checks |
| 9 | AWDL timing precision insufficient on commodity hardware | **MEDIUM** | 🧪 EXPERIMENTAL — needs testing |
| 10 | Cross-platform Rust+JNI/FFI bridge introduces instability | **MEDIUM** | 🟡 PROBABLE — well-known challenge |

---

## 3. Top 10 Research Questions

| # | Question | Current Answer | Confidence |
|---|----------|---------------|------------|
| 1 | What is the minimum implementation to appear in iPhone's AirDrop list? | BLE advert + AWDL sync + mDNS service + /Discover response | 🟡 PROBABLE |
| 2 | Does iOS 18+ validate BLE advertisement fields beyond Company ID? | Unknown — needs real-device testing | ❓ UNKNOWN |
| 3 | Can self-signed TLS certs still work with iOS 18 "Everyone" mode? | Yes, per opendrop-rs validation | ✅ CONFIRMED |
| 4 | What happens when AW timing drifts on non-Apple hardware? | Degraded connectivity; needs empirical measurement | 🧪 EXPERIMENTAL |
| 5 | When will Apple ship Wi-Fi Aware (NAN) for AirDrop (DMA)? | 2026–2027 based on regulatory pressure | 🟡 PROBABLE |
| 6 | Can Android's Wi-Fi Aware API interoperate with Apple's AirDrop? | Not yet; would require Apple to support NAN for AirDrop | ❓ UNKNOWN |
| 7 | What ReceiverModelName should we spoof for maximum compatibility? | "MacBookPro18,1" or similar | 🧪 EXPERIMENTAL |
| 8 | Does iOS timeout /Ask → /Upload if too slow? What's the timeout? | Yes, ~30 seconds estimated | 🧪 EXPERIMENTAL |
| 9 | Can we achieve > 10 MB/s throughput over AWDL with commodity hardware? | ~5-10 MB/s reported; half-duplex limits throughput | 🟡 PROBABLE |
| 10 | Is the CPIO archive format consistent across iOS versions? | Likely stable but needs version testing | 🟡 PROBABLE |

---

## 4. First 20 Implementation Tasks

| # | Task | Phase | Priority | Estimated Effort |
|---|------|-------|----------|-----------------|
| 1 | Acquire and verify Wi-Fi adapter (monitor mode + injection) | 1 | 🔴 CRITICAL | 1 day |
| 2 | Set up Linux development environment (Ubuntu 24.04 + tools) | 1 | 🔴 CRITICAL | 1 day |
| 3 | Capture iPhone AWDL frames with Wireshark | 1 | 🔴 CRITICAL | 1 day |
| 4 | Capture iPhone BLE AirDrop advertisement | 1 | 🔴 CRITICAL | 1 day |
| 5 | Initialize Cargo workspace with crate stubs | 1 | HIGH | 0.5 day |
| 6 | Implement `ud-core`: error types, logging (tracing), config | 1 | HIGH | 1 day |
| 7 | Implement `ud-crypto`: Ed25519 keygen, self-signed cert gen | 1 | HIGH | 1 day |
| 8 | Build BLE scanner tool (decode Apple advertisements) | 1 | HIGH | 2 days |
| 9 | Document captured packet formats (annotated hex + diagrams) | 1 | HIGH | 1 day |
| 10 | Set up CI (GitHub Actions: build, test, clippy, fmt, cargo deny) | 1 | MEDIUM | 0.5 day |
| 11 | Implement AWDL action frame parser (TLV decoding) | 2 | 🔴 CRITICAL | 3 days |
| 12 | Implement AWDL frame generator | 2 | 🔴 CRITICAL | 2 days |
| 13 | Implement AWDL channel hopper (tune to social channels) | 2 | 🔴 CRITICAL | 2 days |
| 14 | Implement AWDL sync (parse PSF, align to master) | 2 | 🔴 CRITICAL | 3 days |
| 15 | Implement AWDL election (participate with low priority) | 2 | HIGH | 1 day |
| 16 | Create AWDL virtual interface (IPv6 link-local) | 2 | HIGH | 2 days |
| 17 | Implement mDNS _airdrop._tcp service on AWDL interface | 2 | HIGH | 1 day |
| 18 | Implement HTTPS server with self-signed cert on AWDL | 2 | HIGH | 2 days |
| 19 | Implement /Discover handler (binary plist request/response) | 2 | HIGH | 2 days |
| 20 | Test: iPhone discovers linux-receiver in AirDrop list | 2 | 🔴 CRITICAL | 1 day |

---

## 5. First 10 Git Commits

```
1. feat: initialize cargo workspace with crate stubs
   - Cargo.toml workspace
   - ud-core, ud-crypto, ud-protocol, ud-transfer, ud-airdrop, ud-awdl, ud-ble, ud-mdns stubs
   - LICENSE-APACHE, LICENSE-MIT, README.md

2. feat(ud-core): add error types, structured logging, config
   - Error enum with categories
   - tracing integration with log categories ([BLE], [AWDL], etc.)
   - Configuration struct with serde

3. feat(ud-crypto): key generation and certificate creation
   - Ed25519 keypair generation
   - Self-signed X.509 certificate from keypair
   - RSA 2048-bit self-signed cert (for AirDrop compat)
   - Unit tests for key/cert round-trips

4. feat(tools): BLE scanner for Apple advertisements
   - Scan and decode BLE advertisements
   - Filter for Apple Company ID 0x004C
   - Decode AirDrop type 0x05 payload
   - Pretty-print results

5. feat(ud-awdl): AWDL action frame parser
   - Parse MIF, PSF, and election TLVs
   - Validate frame structure
   - Unit tests with captured frame data

6. feat(ud-awdl): AWDL frame generator and channel management
   - Generate AWDL action frames
   - Channel hopping to social channels
   - Monitor mode interface management

7. feat(ud-awdl): synchronization and election
   - Parse PSF timing parameters
   - Sync local clock to master
   - Master election participation (low priority)
   - Availability window scheduling

8. feat(ud-mdns): mDNS service advertisement
   - Register _airdrop._tcp service
   - Handle mDNS queries
   - Bind to AWDL interface

9. feat(ud-airdrop): HTTPS server and /Discover handler
   - TLS server with self-signed cert
   - Binary plist parser (request)
   - Binary plist generator (response)
   - /Discover endpoint handler

10. feat(apps/linux-receiver): integrated AirDrop receiver CLI
    - Wire all components
    - CLI arguments (--interface, --save-dir, --log-level)
    - Structured logging output
    - BLE advertising
    - AWDL sync + mDNS + /Discover
```

---

## 6. Recommended Repository Structure

```
universaldrop/
│
├── Cargo.toml                      # Workspace root
├── LICENSE-APACHE                  # Apache-2.0 license
├── LICENSE-MIT                     # MIT license
├── THIRD_PARTY_NOTICES             # Third-party attributions
├── README.md                       # Project overview
│
├── crates/
│   ├── ud-core/                    # Shared types, error handling, logging, config
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── error.rs
│   │       ├── config.rs
│   │       └── logging.rs
│   │
│   ├── ud-crypto/                  # Cryptographic primitives
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── keys.rs             # Ed25519 key management
│   │       ├── certs.rs            # X.509 certificate generation
│   │       └── tls.rs              # TLS configuration
│   │
│   ├── ud-protocol/                # Native UniversalDrop protocol
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── messages.rs         # Message types and serialization
│   │       ├── header.rs           # Message header
│   │       └── version.rs          # Version negotiation
│   │
│   ├── ud-transfer/                # File transfer engine
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── chunker.rs          # File chunking
│   │       ├── resume.rs           # Resume state management
│   │       ├── integrity.rs        # SHA-256 hash verification
│   │       ├── sanitize.rs         # File name sanitization
│   │       └── cpio.rs             # CPIO archive parser
│   │
│   ├── ud-airdrop/                 # AirDrop application protocol
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── discover.rs         # /Discover handler
│   │       ├── ask.rs              # /Ask handler
│   │       ├── upload.rs           # /Upload handler
│   │       ├── plist.rs            # Binary plist utilities
│   │       └── server.rs           # HTTPS server
│   │
│   ├── ud-awdl/                    # AWDL link layer
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── frames.rs           # Frame parsing/generation
│   │       ├── tlv.rs              # TLV parser
│   │       ├── sync.rs             # Synchronization
│   │       ├── election.rs         # Master election
│   │       ├── channel.rs          # Channel management
│   │       └── interface.rs        # Virtual interface
│   │
│   ├── ud-ble/                     # BLE advertising/scanning
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── apple.rs            # Apple BLE advertisement format
│   │       ├── advertise.rs        # BLE advertising
│   │       └── scan.rs             # BLE scanning
│   │
│   └── ud-mdns/                    # mDNS service discovery
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs
│           ├── service.rs          # Service registration
│           └── discovery.rs        # Service discovery
│
├── apps/
│   ├── linux-receiver/             # Linux CLI AirDrop receiver
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── main.rs
│   │
│   ├── universaldrop-cli/          # Cross-platform CLI (native protocol)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── main.rs
│   │
│   ├── android/                    # Android app (Kotlin + Rust)
│   │   ├── app/
│   │   ├── rust/                   # JNI bridge
│   │   ├── build.gradle.kts
│   │   └── settings.gradle.kts
│   │
│   └── windows/                    # Windows app (Rust + egui)
│       ├── Cargo.toml
│       └── src/
│           └── main.rs
│
├── tools/
│   ├── ble-scanner/                # BLE analysis tool
│   │   ├── Cargo.toml
│   │   └── src/main.rs
│   │
│   ├── awdl-monitor/               # AWDL traffic monitor
│   │   ├── Cargo.toml
│   │   └── src/main.rs
│   │
│   └── packet-analyzer/            # Protocol packet analyzer
│       ├── Cargo.toml
│       └── src/main.rs
│
├── tests/
│   ├── unit/                       # Additional unit tests
│   ├── integration/                # Cross-crate integration tests
│   └── fixtures/                   # Test data (captured packets, sample plists)
│
├── docs/                           # Documentation (this DOC folder)
│   ├── executive-summary.md
│   ├── feasibility-report.md
│   ├── phase-plan.md
│   ├── airdrop-protocol.md
│   ├── awdl-research.md
│   ├── security-threat-model.md
│   ├── test-strategy.md
│   ├── license-audit.md
│   ├── hardware-compatibility.md
│   ├── native-protocol-design.md
│   ├── risk-register.md
│   ├── starting-point.md
│   ├── adr/
│   │   └── decisions.md
│   └── captures/                   # Packet capture documentation
│       ├── ble/
│       ├── awdl/
│       └── airdrop/
│
└── .github/
    └── workflows/
        ├── ci.yml                  # Build, test, lint
        └── license-check.yml       # cargo deny
```

---

## 7. Recommended License Strategy

```
License:            Apache-2.0 OR MIT (dual license)
Rationale:          Rust ecosystem standard; maximum compatibility
Dependencies:       All permissively licensed (verified)
GPLv3 avoidance:    Independent implementation; no code from OpenDrop/OWL/opendrop-rs
Patent risk:        Acknowledged; consult legal for commercial distribution
CI enforcement:     cargo deny configured to reject copyleft dependencies
Attribution:        THIRD_PARTY_NOTICES maintained for all dependencies
```

---

## 8. MVP Definitions

### MVP 1 — Linux AirDrop Receiver ✅ (Phases 1-3)
```
iPhone → AirDrop → Linux → file received
```

### MVP 2 — Native Cross-Platform (Phases 4-6)
```
Android ↔ UniversalDrop ↔ Windows
```

### MVP 3 — Android AirDrop Receiver (Phase 7)
```
iPhone → AirDrop → Android → file received
(Requires Wi-Fi Aware / DMA or hardware-level AWDL support)
```

### MVP 4 — Windows AirDrop Bridge (Phase 8)
```
iPhone → AirDrop → [Linux bridge] → Windows → file received
```

### MVP 5 — Full Product (Phase 12)
```
iPhone ↔ Android ↔ Windows
via AirDrop compatibility + native UniversalDrop protocol
```

---

## 9. Minimum Implementation for iPhone Discovery

### Required (for iPhone to discover and send)
```
✅ BLE advertisement with Apple Company ID 0x004C, type 0x05
✅ AWDL link layer synchronized to iPhone's AWDL cluster
✅ mDNS _airdrop._tcp service registered on AWDL interface
✅ HTTPS server listening on advertised port
✅ /Discover handler returning valid binary plist response
✅ /Ask handler accepting transfer request
✅ /Upload handler receiving CPIO archive
✅ Self-signed TLS certificate (for "Everyone" mode)
```

### Optional
```
🟡 Apple-signed identity record (only for "Contacts Only" mode)
🟡 ReceiverMediaCapabilities (for format conversion requests)
🟡 Icon/thumbnail in /Ask response
```

### Only Required for Sending TO Apple Devices
```
⬜ Apple-signed SenderRecordData
⬜ Valid Apple ID validation record
```

### Currently Impossible (without Apple)
```
🔴 "Contacts Only" mode discovery/authentication
🔴 Web link sharing (requires Apple identity)
🔴 Apple-signed device identity
```
