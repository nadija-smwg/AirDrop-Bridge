# UniversalDrop — Architecture Decision Records (ADRs)

**Date:** September 2026

---

## ADR-001: Core Language — Rust

### Context
UniversalDrop needs a language that works across Linux, Android (via NDK), and Windows with high performance, memory safety, and excellent networking support.

### Options
1. **Rust** — Memory-safe, cross-platform, excellent async networking (tokio), FFI to Android/Windows
2. **C++** — Maximum performance, but memory safety concerns, complex build systems
3. **Go** — Good networking, but poor mobile support, large binary size, no Android NDK integration
4. **Python** — Rapid prototyping (OpenDrop uses it), but poor performance, hard to deploy on mobile

### Decision
**Rust**

### Reason
- Memory safety without garbage collection — critical for a security-sensitive networking application
- `tokio` provides world-class async I/O
- `rustls` provides a pure-Rust TLS implementation (no OpenSSL dependency)
- Android NDK support via `android-ndk-rs`
- Windows support via `windows-rs` crate
- Cargo workspaces enable clean modular architecture
- Strong ecosystem for serialization (`serde`), crypto (`ring`), and BLE (`btleplug`)
- The most mature existing implementation (opendrop-rs) is already in Rust

### Trade-offs
- Steeper learning curve than Python/Go
- Longer compilation times
- Android UI still requires Kotlin (Rust via JNI bridge)
- Windows UI may require C#/XAML (Rust core via FFI)

### Consequences
- All core protocol code in Rust
- Platform-specific UI layers in native languages (Kotlin, C#/Rust)
- Must maintain JNI/FFI bridges

---

## ADR-002: Transfer Protocol — TCP + TLS 1.3

### Context
The native UniversalDrop protocol needs a reliable, encrypted transport for file transfers over LAN.

### Options
1. **TCP + TLS 1.3** — Standard, reliable, well-debugged
2. **QUIC** — Modern, multiplexed, built-in encryption
3. **Raw TCP** — Simple but no encryption
4. **UDP** — Low overhead but unreliable

### Decision
**TCP + TLS 1.3** for initial implementation. QUIC as future option.

### Reason
- TCP+TLS is universally supported and debuggable
- LAN transfers don't benefit significantly from QUIC's NAT traversal
- File transfer is primarily sequential — multiplexing benefit is minimal
- `rustls` + `tokio` provide a mature, well-tested stack
- QUIC adds complexity without proportional LAN benefit
- TCP+TLS debugging is more accessible (Wireshark, tcpdump)

### Trade-offs
- No 0-RTT reconnection (QUIC advantage)
- No built-in stream multiplexing
- Slightly higher handshake latency than QUIC for repeat connections

### Consequences
- Transport layer abstracted behind trait for future QUIC support
- Protocol messages are transport-agnostic
- QUIC (via `quinn` crate) can be added as alternative transport later

---

## ADR-003: Discovery Architecture — mDNS + Optional BLE

### Context
Devices need to discover each other on the local network.

### Options
1. **mDNS (Bonjour/DNS-SD)** — Standard local network discovery
2. **BLE + mDNS** — BLE for proximity, mDNS for service resolution
3. **UDP broadcast** — Simple but limited
4. **Manual IP entry** — Reliable but poor UX

### Decision
**mDNS as primary discovery, BLE as optional proximity signal**

### Reason
- mDNS works on all platforms (Linux: Avahi, Android: NsdManager, Windows: native)
- mDNS is the standard for local network service discovery
- BLE provides proximity awareness but isn't required for LAN transfers
- AirDrop compatibility requires BLE anyway, so the infrastructure will exist

### Trade-offs
- mDNS only works on the same subnet
- BLE adds complexity and permission requirements

### Consequences
- `_universaldrop._tcp.local.` service registration
- BLE advertising optional (used when AirDrop module is active)

---

## ADR-004: AWDL Strategy — Independent Reimplementation

### Context
AirDrop requires AWDL, which has been reverse-engineered by SEEMOO Lab (OWL, GPLv3). We need an implementation.

### Options
1. **Independent Rust reimplementation** — Clean-room, permissive license
2. **Fork OWL (GPLv3)** — Fast but forces GPLv3 on entire project
3. **Use OWL as subprocess** — IPC bridge, isolates GPLv3
4. **Skip AWDL** — Wait for Wi-Fi Aware (DMA)

### Decision
**Independent Rust reimplementation** (Option A), with **OWL as temporary research fallback** (Option C) during development.

### Reason
- GPLv3 on the entire project would limit adoption and commercial use
- Independent implementation allows Apache-2.0/MIT licensing
- The AWDL protocol is publicly documented through academic papers
- opendrop-rs (also GPLv3) provides architecture reference without code copying
- OWL subprocess can be used temporarily for validation during research

### Trade-offs
- Significantly more development effort
- Risk of implementation bugs not present in OWL
- Must be very careful not to accidentally copy GPLv3 code

### Consequences
- `ud-awdl` crate is an independent implementation
- Protocol behavior validated against captured packets, not against OWL source
- License audit required before each release

---

## ADR-005: Android Architecture — Kotlin UI + Rust Core (JNI)

### Context
Android app needs native UI and access to Rust core library.

### Options
1. **Kotlin + Rust via JNI** — Native UI, shared core
2. **Flutter + Rust via FFI** — Cross-platform UI, shared core
3. **Pure Kotlin** — Simple but duplicates protocol implementation
4. **React Native + Rust** — JS UI, shared core

### Decision
**Kotlin + Jetpack Compose UI with Rust core via JNI/NDK**

### Reason
- Native Android UI provides best performance and platform integration
- Jetpack Compose is the modern Android UI framework
- Rust core ensures protocol consistency across platforms
- JNI bridge is well-established for Android NDK
- Native UI integrates better with Android features (notifications, share intents, foreground services)

### Trade-offs
- Must maintain JNI bridge
- Can't reuse UI across platforms (each platform has native UI)
- JNI has overhead and debugging complexity

### Consequences
- Android app in `apps/android/`
- Rust core compiled for Android targets (aarch64, armv7, x86_64)
- JNI bindings auto-generated where possible (`jni` crate)
- UI follows Material 3 guidelines

---

## ADR-006: Windows Architecture — Rust Core + Native UI

### Context
Windows app needs modern UI and Rust core library access.

### Options
1. **Rust + egui** — Pure Rust, cross-platform GUI
2. **Rust + WinUI 3 (C#)** — Native Windows UI, Rust backend
3. **Rust + Tauri** — Web-based UI, Rust backend
4. **Pure C# (.NET)** — Native but duplicates protocol

### Decision
**Rust core + egui for initial version; evaluate WinUI 3 for production**

### Reason
- egui provides rapid development entirely in Rust
- No C# bridge needed initially
- WinUI 3 can be adopted later for polish
- egui is sufficient for MVP
- Tauri adds web stack complexity

### Trade-offs
- egui UI looks less "Windows-native" than WinUI 3
- WinUI 3 integration requires C#/Rust interop
- egui has limited Windows integration (system tray, etc.)

### Consequences
- Initial Windows app uses egui
- Production version may migrate to WinUI 3
- Windows Service for background operation regardless of UI choice

---

## ADR-007: Security Architecture — TOFU + Ed25519

### Context
Devices need to authenticate each other without a central authority.

### Options
1. **TOFU (Trust On First Use)** — Pin certificate on first encounter
2. **Pre-shared key** — Exchange key out-of-band before first use
3. **Central CA** — Server-based certificate authority
4. **No authentication** — Rely on TLS encryption only

### Decision
**TOFU with Ed25519 device identity**

### Reason
- No central server required (peer-to-peer architecture)
- Similar to SSH's known_hosts model — well-understood security properties
- Ed25519 is fast, compact, and well-supported in Rust
- Users can visually verify fingerprints on first connection
- Pinned certificates prevent subsequent MITM attacks

### Trade-offs
- First connection is vulnerable to MITM (inherent to TOFU)
- Users must trust the initial connection
- Certificate changes require user re-verification

### Consequences
- Each device generates Ed25519 keypair on first launch
- Self-signed X.509 certificate derived from keypair
- Certificate fingerprint displayed to user on first pairing
- Pinned certificates stored in local database

---

## ADR-008: Storage Architecture — Scoped/Sandboxed

### Context
Received files must be stored safely on each platform.

### Options
1. **Designated "UniversalDrop" folder** — Simple, user-accessible
2. **OS Downloads folder** — Standard location
3. **App-private storage** — Secure but hard to access
4. **User-configurable** — Flexible

### Decision
**Default to "UniversalDrop" subfolder in OS Downloads; user-configurable**

### Reason
- Downloads folder is the standard location for received files
- Subfolder prevents clutter in main Downloads
- User-configurable for advanced users
- Android: works with scoped storage (MediaStore or SAF)

### Trade-offs
- Subfolder may not be what all users expect
- Android scoped storage adds complexity

### Consequences
- Default: `~/Downloads/UniversalDrop/` (Linux), `Downloads\UniversalDrop\` (Windows), app-designated directory (Android)
- Temp files written to app-private temp directory
- Atomic move from temp to final location

---

## ADR-009: Licensing — Apache-2.0 OR MIT (Dual License)

### Context
Project license must be compatible with Rust ecosystem and allow broad adoption.

### Decision
**Apache-2.0 OR MIT dual license** (standard Rust ecosystem convention)

### Reason
- Compatible with all Rust crate dependencies
- Allows commercial use without copyleft obligations
- Standard practice in the Rust community
- Not contaminated by GPLv3 (independent implementation)

### Consequences
- LICENSE-APACHE and LICENSE-MIT files in repository root
- THIRD_PARTY_NOTICES listing all dependency licenses
- No code copied from GPLv3 projects (OpenDrop, OWL, opendrop-rs)

---

## ADR-010: AirDrop Compatibility Strategy — "Everyone" Mode Only

### Context
AirDrop has two discovery modes: "Contacts Only" and "Everyone." "Contacts Only" requires Apple-signed identity certificates.

### Decision
**Support "Everyone" mode only for AirDrop compatibility**

### Reason
- "Everyone" mode is the only mode that works without Apple-signed credentials
- This is confirmed by all existing open implementations (OpenDrop, opendrop-rs)
- "Contacts Only" requires cryptographic material that only Apple can issue
- The DMA may eventually provide a standards-based alternative

### Trade-offs
- Users must set their iPhone to "Everyone" or "Everyone for 10 minutes"
- iPhone auto-reverts to "Contacts Only" after 10 minutes
- Web link sharing doesn't work (requires Apple identity)
- Cannot impersonate a contact's device

### Consequences
- Documentation clearly states "Everyone mode" requirement
- UI guides user to set iPhone to "Everyone for 10 minutes"
- No attempt to reproduce Apple identity certificates
- Monitor DMA developments for future "Contacts Only" path
