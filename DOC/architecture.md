# UniversalDrop — Architecture Overview

**Date:** September 2026

---

## 1. System Architecture

```
                          UNIVERSALDROP
                               │
                  ┌────────────┼────────────┐
                  │                         │
                  ▼                         ▼
          AIRDROP MODULE              NATIVE MODULE
          (iPhone interop)            (Android/Windows)
                  │                         │
          ┌───────┼───────┐         ┌───────┼───────┐
          │       │       │         │       │       │
          ▼       ▼       ▼         ▼       ▼       ▼
        BLE    AWDL   AirDrop     mDNS   TCP/TLS  Protocol
       Advert  Link   HTTP/TLS   Disc   Transport  Messages
          │       │       │         │       │       │
          └───────┼───────┘         └───────┼───────┘
                  │                         │
                  └────────────┬────────────┘
                               │
                  ┌────────────┼────────────┐
                  │                         │
                  ▼                         ▼
           TRANSFER CORE            SECURITY CORE
           • Chunking               • Keys (Ed25519)
           • Resume                 • Certificates
           • Integrity (SHA-256)    • TLS Config
           • File I/O               • Auth
           • Progress               • File Safety
           • CPIO Parser            • Sanitization
                  │                         │
                  └────────────┬────────────┘
                               │
                        PLATFORM LAYER
                        • OS-specific APIs
                        • Storage
                        • Notifications
                        • Permissions
                        • Credential Store
                               │
                  ┌────────────┼────────────┐
                  │            │            │
                  ▼            ▼            ▼
              ANDROID       WINDOWS       LINUX
               App           App         CLI/Daemon
              (Kotlin)      (Rust/egui)   (Rust)
```

---

## 2. Crate Dependency Graph

```
                    ud-core
                   /  |  \
                  /   |   \
           ud-crypto  |  ud-protocol
              |       |       |
              |   ud-transfer |
              |    /  |  \    |
              |   /   |   \   |
           ud-ble  ud-awdl  ud-airdrop
              |       |       |
              |   ud-mdns    |
              |       |       |
              └───────┼───────┘
                      │
               linux-receiver
               universaldrop-cli
               android (via JNI)
               windows (via FFI)
```

---

## 3. Data Flow — AirDrop Receive

```
iPhone                    UniversalDrop (Linux/Android/Windows)
                          
                          [ud-ble]
  ←── BLE Scan ─────────  BLE Advertisement (Apple 0x004C)
                          
                          [ud-awdl]
  ──► AWDL Activate ────  AWDL Sync (channel hop, master election)
                          
                          [ud-mdns]
  ──► mDNS Query ───────  _airdrop._tcp response
                          
                          [ud-airdrop + ud-crypto]
  ──► TLS Handshake ────  Self-signed TLS cert
                          
  ──► POST /Discover ───  Parse plist, respond with receiver info
                          
  ──► POST /Ask ────────  Parse file metadata, show accept dialog
                          
                          [ud-transfer]
  ──► POST /Upload ─────  Receive CPIO, extract, verify, save
```

---

## 4. Data Flow — Native Protocol Transfer

```
Device A (Sender)          Device B (Receiver)

                           [ud-mdns]
  ←── mDNS Discovery ───  _universaldrop._tcp
                           
                           [ud-crypto]
  ──► TCP + TLS 1.3 ────  Certificate exchange
                           
                           [ud-protocol]
  ──► HELLO ─────────────  Version + capability negotiation
  ◄── HELLO ──────────── 
                           
  ──► DEVICE_INFO ───────  Device metadata exchange
  ◄── DEVICE_INFO ──────
                           
  ──► TRANSFER_REQUEST ──  File list with sizes and hashes
  ◄── TRANSFER_ACCEPT ──
                           
                           [ud-transfer]
  ──► FILE_METADATA ─────  Per-file info
  ──► FILE_CHUNK(0) ─────  1 MB chunk
  ◄── FILE_ACK(0) ──────
  ──► FILE_CHUNK(1) ─────  1 MB chunk
  ◄── FILE_ACK(1) ──────
  ... (repeat) ...
  ──► FILE_COMPLETE ─────  
  ◄── HASH_VERIFY ──────  SHA-256 confirmation
  ◄── TRANSFER_COMPLETE ─  Done ✓
```

---

## 5. Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | Rust | Memory safety, cross-platform, performance |
| **Async Runtime** | tokio | Industry-standard async I/O |
| **TLS** | rustls | Pure-Rust, no OpenSSL |
| **Crypto** | ring + ed25519-dalek | Well-audited, fast |
| **Serialization** | serde + custom binary | Type-safe, efficient |
| **BLE** | btleplug | Cross-platform BLE |
| **Packets** | pcap crate + libpcap | AWDL frame capture/injection |
| **mDNS** | mdns-sd | Service discovery |
| **Plist** | plist crate | Apple binary plist parsing |
| **Logging** | tracing | Structured, async-compatible |
| **CLI** | clap | Argument parsing |
| **Android UI** | Kotlin + Jetpack Compose | Native Material 3 |
| **Windows UI** | egui → WinUI 3 | Pure Rust MVP → native production |
| **Build** | Cargo workspaces | Monorepo with shared core |
| **CI** | GitHub Actions | Build, test, lint, license check |

---

## 6. Reuse/Reimplement Decision Matrix

| Component | Strategy | Reason |
|-----------|----------|--------|
| AWDL link layer | **A: Independent reimplementation** | GPLv3 (OWL); protocol documented in papers |
| AirDrop HTTP protocol | **A: Independent reimplementation** | GPLv3 (OpenDrop); protocol well-understood |
| BLE advertising | **B: Use btleplug (MIT)** | Permissive license; cross-platform |
| TLS | **B: Use rustls (Apache/MIT)** | Best-in-class Rust TLS |
| Binary plist | **B: Use plist crate (MIT)** | Well-maintained; correct |
| mDNS | **B: Use mdns-sd (Apache/MIT)** | Permissive; cross-platform |
| Crypto | **B: Use ring/ed25519-dalek** | Audited; permissive |
| CPIO parser | **A: Independent implementation** | Simple format; avoids deps |
| Native protocol | **A: Independent design** | Novel protocol; no existing impl |
| File transfer | **A: Independent implementation** | Custom requirements (resume, chunking) |
