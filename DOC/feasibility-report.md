# UniversalDrop — Feasibility Report

**Date:** September 2026
**Research Methodology:** Web research, repository analysis, academic paper review, API documentation review

---

## 1. Final Goal

```
iPhone → AirDrop → UniversalDrop → Android / Windows
Android ↔ UniversalDrop ↔ Windows
```

The iPhone path must use genuine AirDrop protocol interoperability — no QR codes, browser uploads, cloud intermediaries, or companion iOS apps.

---

## 2. Current State of AirDrop Interoperability

### 2.1 What Has Been Proven

| Capability | Status | Evidence |
|-----------|--------|----------|
| Full AirDrop protocol reverse-engineered | ✅ CONFIRMED | SEEMOO Lab published comprehensive analysis; CISPA published 7-layer state machine reconstruction (IEEE WOOT 2026) |
| iPhone → Linux file transfer | ✅ CONFIRMED | `opendrop-rs`/`luftlift-rs` validated on iOS 18 real devices |
| Linux → iPhone file transfer | 🧪 EXPERIMENTAL | OpenDrop (Python) has send capability; less reliable |
| "Everyone" mode works with open implementations | ✅ CONFIRMED | All implementations report success with "Everyone for 10 minutes" |
| "Contacts Only" mode | 🔴 NOT FEASIBLE without Apple certs | Requires Apple-signed identity records; cannot be reproduced |
| Google Pixel 10 has native AirDrop interop | ✅ CONFIRMED | Launched November 2025; reverse-engineered AWDL implementation |
| Samsung/OnePlus/Xiaomi gaining support | 🟡 PROBABLE | Rollout throughout 2026 on flagship devices with compatible hardware |
| Apple moving toward Wi-Fi Aware (DMA) | ✅ CONFIRMED | EU Digital Markets Act Article 6(7) specification proceedings |

### 2.2 Current AirDrop Protocol Behavior (2026)

Based on cross-referencing research:

1. **BLE Discovery:** iPhones broadcast BLE advertisements with Apple Company ID `0x004C`, containing short identity hashes derived from Apple Account contact info
2. **AWDL Activation:** Upon discovering a potential peer via BLE, devices activate AWDL to establish a direct peer-to-peer Wi-Fi link
3. **mDNS Service Discovery:** `_airdrop._tcp` service is advertised over AWDL
4. **HTTPS Application Protocol:** `/Discover`, `/Ask`, `/Upload` endpoints over TLS
5. **Binary Plist Encoding:** All metadata exchanged as binary property lists
6. **CPIO Archives:** File payloads bundled as CPIO archives with DVZip compression
7. **Session Constraint:** `/Ask` and `/Upload` must occur within the same TLS session

### 2.3 Key Changes in Recent iOS Versions

- **"Everyone for 10 minutes" timeout:** Apple now auto-reverts to "Contacts Only" after 10 minutes
- **One-time AirDrop codes:** Some transfer scenarios require manual verification codes
- **Protocol hardening:** Additional validation on HTTP parsing and plist processing
- **Rotating BLE MAC addresses:** Enhanced privacy makes long-term device tracking harder
- **Shared web links require Apple identity:** Cannot transfer URLs to unauthenticated receivers

---

## 3. Existing Open-Source Projects

### 3.1 Comparison Table

| Project | Repository | Language | License | Last Activity | AWDL | AirDrop | iOS Tested | Android | Windows | Hardware Req | Status | Reusable |
|---------|-----------|----------|---------|---------------|------|---------|------------|---------|---------|-------------|--------|----------|
| **OpenDrop** | seemoo-lab/opendrop | Python | **GPLv3** | ~April 2021 | Via OWL | ✅ Send/Receive | Yes (older) | ❌ | ❌ | Monitor-mode Wi-Fi | ⚠️ Unmaintained | Reference only (GPLv3) |
| **OWL** | seemoo-lab/owl | C | **GPLv3** | Active (in Kali) | ✅ Full | N/A (link layer) | Via OpenDrop | ❌ | ❌ | Monitor-mode Wi-Fi | ⚠️ Experimental | Reference only (GPLv3) |
| **opendrop-rs** | ayourtch-llm/opendrop-rs | Rust | **GPLv3** | Recent | ✅ (`filin-rs`) | ✅ (`luftlift-rs`) | iOS 18 ✅ | ❌ | ❌ | Monitor-mode Wi-Fi | ✅ Active, 360+ tests | Reference only (GPLv3) |
| **LocalSend** | localsend/localsend | Dart/Flutter | MIT | Active | ❌ | ❌ | N/A | ✅ | ✅ | Standard Wi-Fi | ✅ Production | Protocol design reference |

### 3.2 Detailed Project Analysis

#### OpenDrop (seemoo-lab/opendrop)
- **Status:** ⚠️ Unmaintained — last release April 2021, issues restricted
- **Value:** The original reverse-engineering reference for AirDrop protocol
- **Limitation:** Python-based, slow, unreliable with newer iOS versions
- **License Impact:** GPLv3 — cannot copy code into a permissively licensed project
- **Reuse Strategy:** Study protocol behavior; reimplement independently in Rust

#### OWL (seemoo-lab/owl)
- **Status:** ⚠️ Experimental but referenced in Kali Linux
- **Value:** The foundational AWDL userspace implementation
- **Architecture:** Uses Netlink API + libpcap for frame injection/reception
- **Limitation:** Requires active monitor mode Wi-Fi hardware
- **License Impact:** GPLv3 — same restriction as OpenDrop
- **Reuse Strategy:** Study AWDL behavior; reimplement independently in Rust

#### opendrop-rs / luftlift-rs (ayourtch-llm/opendrop-rs)
- **Status:** ✅ Most promising — recent activity, validated on iOS 18
- **Architecture:** Two binaries:
  - `filin-rs` — AWDL link layer (monitor interface, election/sync, frame bridging)
  - `luftlift-rs` — AirDrop application layer (mDNS, TLS/HTTPS, /Discover, /Ask, /Upload)
- **Limitation:** No BLE advertising (iPhone must already be looking), large uploads can stall
- **License Impact:** GPLv3 (derived from OWL/OpenDrop)
- **Reuse Strategy:** Primary architecture reference; reimplement independently

#### LocalSend
- **Status:** ✅ Production-quality, widely adopted
- **Value:** Not AirDrop compatible — but excellent reference for native cross-platform file transfer protocol design
- **License:** MIT — freely reusable
- **Reuse Strategy:** Study protocol design patterns for the Native UniversalDrop protocol

---

## 4. AirDrop Architecture

### 4.1 Complete Communication Stack

```
┌──────────────────────────────────────────────────────┐
│                    APPLICATION                        │
│  /Discover → /Ask → /Upload (HTTPS, binary plist)    │
├──────────────────────────────────────────────────────┤
│                    SECURITY                           │
│  TLS 1.2+ with self-signed certs / Apple identity    │
├──────────────────────────────────────────────────────┤
│                 SERVICE DISCOVERY                     │
│  mDNS / Bonjour: _airdrop._tcp                      │
├──────────────────────────────────────────────────────┤
│               PEER-TO-PEER LINK                      │
│  AWDL (Apple Wireless Direct Link)                   │
│  Channel hopping, availability windows, sync         │
├──────────────────────────────────────────────────────┤
│                  DISCOVERY                            │
│  BLE advertisements: Apple Company ID 0x004C         │
│  Short identity hash, AWDL state info                │
├──────────────────────────────────────────────────────┤
│                  PHYSICAL                             │
│  Bluetooth 4.0+ (BLE) + Wi-Fi (2.4/5 GHz)           │
└──────────────────────────────────────────────────────┘
```

### 4.2 Transfer Sequence

```
iPhone (Sender)                    UniversalDrop (Receiver)
       │                                    │
       │── BLE Scan ──────────────────────► │
       │                    ◄── BLE Advert ─│ (Apple 0x004C, type 0x05)
       │                                    │
       │── AWDL Activate ─────────────────► │
       │     Channel sync, master election  │
       │                                    │
       │── mDNS Query ───────────────────►  │
       │   _airdrop._tcp                    │
       │                    ◄── mDNS Resp ──│ (service record)
       │                                    │
       │── TLS Handshake ────────────────►  │
       │                                    │
       │── POST /Discover ───────────────►  │
       │   {SenderRecordData, ...}          │
       │                    ◄── Response ───│ {ReceiverComputerName, ...}
       │                                    │
       │   [User selects receiver in UI]    │
       │                                    │
       │── POST /Ask ────────────────────►  │
       │   {Files metadata, icon, ...}      │
       │                    ◄── Response ───│ {Accept/Reject}
       │                                    │
       │   [If accepted]                    │
       │                                    │
       │── POST /Upload ─────────────────►  │
       │   {CPIO archive, DVZip compressed} │
       │                    ◄── Response ───│ {Success}
       │                                    │
       │   [Transfer complete]              │
```

---

## 5. BLE Discovery

### 5.1 Purpose
BLE serves as the low-power "paging" mechanism that initiates the AirDrop discovery process.

### 5.2 Technical Details

| Aspect | Detail |
|--------|--------|
| **Transport** | Bluetooth Low Energy advertisements |
| **Company ID** | `0x004C` (Apple, Inc.) |
| **AD Type** | `0xFF` (Manufacturer Specific Data) |
| **AirDrop Type Byte** | `0x05` |
| **Payload** | Short identity hash (truncated hash of contact info) |
| **MAC Addresses** | Rotating (privacy protection) |
| **Scanning** | iPhone scans for compatible BLE advertisements |
| **Advertising** | Receiver advertises availability via BLE |

### 5.3 Receiver Requirements

For the UniversalDrop receiver to be discovered by an iPhone:
- **MUST** broadcast BLE advertisements with Apple Company ID `0x004C`
- **MUST** include AirDrop service type byte `0x05`
- **MUST** include appropriate payload structure

### 5.4 Platform Feasibility

| Platform | BLE Advertising | Apple-format BLE | Status |
|----------|----------------|-----------------|--------|
| **Linux** | ✅ BlueZ / HCI | ✅ Can craft custom payloads | ✅ CONFIRMED works |
| **Android** | ✅ `BLUETOOTH_ADVERTISE` permission | 🧪 Can set manufacturer data | 🧪 EXPERIMENTAL — vendor restrictions possible |
| **Windows** | ✅ `BluetoothLEAdvertisementPublisher` | 🧪 Can set manufacturer data | 🧪 EXPERIMENTAL — need to verify payload flexibility |

### 5.5 Known Limitations
- ❓ **UNKNOWN:** Whether iOS 18+ validates additional BLE fields beyond Company ID and type byte
- 🟡 **PROBABLE:** Some Android vendors restrict custom manufacturer data in BLE advertisements
- ⚠️ **NOTE:** `opendrop-rs` does NOT emit BLE advertisements — it relies on the iPhone already being in "share" mode

---

## 6. AWDL (Apple Wireless Direct Link)

### 6.1 Architecture

AWDL is a proprietary peer-to-peer Wi-Fi protocol operating at the link layer:

```
┌─────────────────────────────────┐
│     AWDL Frame Processing       │
│  (Action frames, data frames)   │
├─────────────────────────────────┤
│     Channel Management          │
│  Social channels: 6, 44, 149   │
│  Availability Windows (AWs)     │
│  Channel hopping / time-slicing │
├─────────────────────────────────┤
│     Synchronization             │
│  Master election (MIF frames)   │
│  Periodic Sync Frames (PSF)     │
│  Clock synchronization          │
├─────────────────────────────────┤
│     Peer Discovery              │
│  Neighbor announcement          │
│  Election metrics               │
├─────────────────────────────────┤
│     IEEE 802.11 (Wi-Fi)         │
│  Monitor mode + injection       │
└─────────────────────────────────┘
```

### 6.2 Key Mechanisms

| Mechanism | Description |
|-----------|-------------|
| **Availability Windows** | Time-sliced windows when device tunes to AWDL social channel |
| **Channel Hopping** | Radio periodically switches between infrastructure Wi-Fi and AWDL social channel |
| **Social Channels** | Channels 6 (2.4 GHz), 44, 149 (5 GHz) |
| **Master Election** | Devices broadcast Master Indication Frames (MIF) with priority metrics |
| **Synchronization** | Slave devices sync clocks to master's Periodic Synchronization Frames (PSF) |
| **Frame Types** | Action frames (management), Data frames (payload) |

### 6.3 Hardware Requirements

| Requirement | Detail | Criticality |
|-------------|--------|-------------|
| **Monitor Mode** | Wi-Fi card MUST support active monitor mode | 🔴 CRITICAL |
| **Packet Injection** | MUST support raw frame injection | 🔴 CRITICAL |
| **5 GHz Support** | Required for social channels 44/149 | ⚠️ HIGH |
| **Channel Switching** | Must support rapid channel changes | ⚠️ HIGH |
| **Concurrent Interfaces** | Ideally support virtual monitor + managed | 🟡 NICE-TO-HAVE |

### 6.4 Chipset Compatibility Matrix

| Chipset | Driver | Monitor Mode | Injection | 5 GHz | AWDL Tested | Status |
|---------|--------|-------------|-----------|-------|-------------|--------|
| **Atheros AR9271** | `ath9k_htc` | ✅ | ✅ | ❌ 2.4GHz only | ✅ OWL | ✅ RECOMMENDED (limited to ch 6) |
| **MediaTek MT7612U** | `mt76` | ✅ | ✅ | ✅ | 🧪 Likely | 🟡 PROMISING |
| **Realtek RTL8812AU** | `rtl8812au` | ✅ | ✅ | ✅ | 🧪 Community reports | 🟡 PROMISING |
| **Intel AX200/210** | `iwlwifi` | ❌ | ❌ | ✅ | ❌ | 🔴 NOT COMPATIBLE |
| **Broadcom** | varies | ❌ mostly | ❌ | varies | ❌ | 🔴 NOT COMPATIBLE |

### 6.5 Platform Feasibility

| Platform | AWDL Feasibility | Mechanism | Status |
|----------|-----------------|-----------|--------|
| **Linux** | ✅ Works | OWL/filin-rs via monitor mode | ✅ CONFIRMED |
| **Android** | 🧪 EXPERIMENTAL | Would require root + custom Wi-Fi driver OR Google's proprietary implementation | Google achieved it for Pixel 10 with custom firmware |
| **Windows** | 🔴 NOT CURRENTLY FEASIBLE | No monitor mode / injection APIs; no userspace AWDL stack | Would require external USB adapter + custom driver |

### 6.6 Critical Analysis

> **AWDL is the single hardest component of the entire project.**

- On **Linux**, it works through userspace implementation with compatible hardware
- On **Android**, Google solved it by working directly with chipset vendors (Qualcomm, etc.) to add firmware-level support — this is not available to third-party developers
- On **Windows**, there is no known path without an external USB Wi-Fi adapter that supports monitor mode under Windows (extremely rare)

---

## 7. mDNS / Bonjour

### 7.1 Purpose
Once AWDL establishes the peer-to-peer link, mDNS (Bonjour) is used to discover the `_airdrop._tcp` service.

### 7.2 Technical Details

| Aspect | Detail |
|--------|--------|
| **Service Type** | `_airdrop._tcp.local.` |
| **Transport** | Multicast DNS over AWDL interface |
| **Port** | Dynamic (advertised in SRV record) |
| **TXT Records** | Device capabilities, flags |
| **Resolution** | Standard mDNS resolution to IPv6 link-local address |

### 7.3 Platform Support

| Platform | mDNS Support | Status |
|----------|-------------|--------|
| **Linux** | ✅ Avahi / manual | ✅ CONFIRMED |
| **Android** | ✅ `NsdManager` API | ✅ CONFIRMED |
| **Windows** | ✅ Native (Win10 1703+) | ✅ CONFIRMED |

### 7.4 Implementation Note
mDNS itself is straightforward and well-supported across all platforms. The challenge is that AirDrop's mDNS operates **over the AWDL interface**, not over standard Wi-Fi — so AWDL must be working first.

---

## 8. AirDrop Application Protocol

### 8.1 Endpoints

#### POST /Discover
| Field | Value |
|-------|-------|
| **Sender** | iPhone |
| **Purpose** | Announce sender; query receiver capabilities |
| **Transport** | HTTPS over AWDL |
| **Encoding** | Binary plist |
| **Key Fields** | `SenderRecordData` (identity), `SenderModelName`, `SenderComputerName` |
| **Response** | `ReceiverComputerName`, `ReceiverModelName`, receiver capabilities |
| **Security** | TLS encrypted; identity hashes compared |

#### POST /Ask
| Field | Value |
|-------|-------|
| **Sender** | iPhone |
| **Purpose** | Request permission to transfer specific files |
| **Transport** | HTTPS over AWDL (same TLS session as /Discover) |
| **Encoding** | Binary plist |
| **Key Fields** | File metadata (name, size, type, UTI), optional icon/preview |
| **Response** | Accept or Reject |
| **User Interaction** | Receiver shows accept/decline dialog |

#### POST /Upload
| Field | Value |
|-------|-------|
| **Sender** | iPhone |
| **Purpose** | Transfer file data |
| **Transport** | HTTPS over AWDL (same TLS session) |
| **Encoding** | CPIO archive, optionally DVZip compressed |
| **Key Fields** | File payload as CPIO archive |
| **Response** | Success/Failure |
| **Constraint** | ⚠️ MUST use same TLS session as /Ask |

### 8.2 Session Constraint
```
CRITICAL: /Ask and /Upload must occur within the SAME TLS session.
Initiating /Upload on a new connection is typically REJECTED by iOS.
```

### 8.3 Binary Plist Structure

AirDrop uses Apple's binary property list format. Key structures:

```
/Discover Request:
{
    "SenderRecordData": <binary>,      // Apple identity record (optional for Everyone)
    "SenderComputerName": "iPhone",
    "SenderModelName": "iPhone15,2",
    "SenderID": <UUID>,
    "BundleID": "com.apple.UIKit.activity.AirDrop"
}

/Discover Response:
{
    "ReceiverComputerName": "UniversalDrop",
    "ReceiverModelName": "MacBookPro",  // Can be spoofed for compatibility
    "ReceiverMediaCapabilities": {...}
}

/Ask Request:
{
    "SenderComputerName": "iPhone",
    "Files": [{
        "FileName": "IMG_4821.HEIC",
        "FileType": "public.heic",
        "FileBomPath": "...",
        "FileIsDirectory": false,
        "ConvertMediaFormats": false
    }],
    "Icon": <thumbnail data>
}
```

---

## 9. Authentication / Identity

### 9.1 Three Distinct Concepts

```
┌────────────────────┐
│ Device Discovery   │ ← BLE advertisements, mDNS
│ "Can I see you?"   │
├────────────────────┤
│ Device Auth        │ ← TLS certificates, Apple identity records
│ "Are you who you   │
│  claim to be?"     │
├────────────────────┤
│ User Authorization │ ← Accept/Decline dialog
│ "Do I want this    │
│  file from you?"   │
└────────────────────┘
```

### 9.2 Identity Mechanism

| Component | Description | Required for UniversalDrop? |
|-----------|-------------|---------------------------|
| **RSA 2048-bit Identity** | Generated when user signs into iCloud | ❌ Receiver doesn't need iCloud |
| **Short Identity Hash** | Truncated hash of contact email/phone | Only for "Contacts Only" matching |
| **Apple-Signed Identity Record** | `SenderRecordData` signed by Apple | ❌ Not required for "Everyone" mode receiving |
| **TLS Certificate** | Self-signed or Apple-signed | ✅ Self-signed works for "Everyone" |
| **Validation Record** | Apple validation of device identity | Only for "Contacts Only" |

### 9.3 What Works Without Apple Credentials

| Feature | Without Apple Certs | Status |
|---------|-------------------|--------|
| **Receive files (Everyone mode)** | ✅ Works | ✅ CONFIRMED by opendrop-rs |
| **Appear in AirDrop list** | ✅ Works | ✅ CONFIRMED |
| **Receive files (Contacts Only)** | ❌ Cannot work | 🔴 Requires Apple-signed identity |
| **Receive web links** | ❌ Cannot work | 🔴 Requires Apple-signed identity |
| **Send files to iPhone** | 🧪 Partial | 🧪 EXPERIMENTAL — iPhone may reject |

### 9.4 Critical Implication

> **UniversalDrop AirDrop compatibility will ONLY work when the iPhone has AirDrop set to "Everyone" or "Everyone for 10 minutes."**

This is a fundamental limitation that cannot be bypassed without Apple's cooperation or the DMA-mandated interoperability path.

---

## 10. TLS / Security

### 10.1 TLS Configuration

| Aspect | Detail |
|--------|--------|
| **Protocol** | TLS 1.2+ |
| **Certificate** | Self-signed RSA 2048-bit acceptable for "Everyone" |
| **Client Auth** | Optional (not enforced for "Everyone") |
| **Session** | Single session for entire Discover→Ask→Upload flow |
| **Cipher** | Standard TLS cipher suites |

### 10.2 Security Considerations for UniversalDrop

| Concern | Mitigation |
|---------|------------|
| MITM on AWDL | TLS encryption prevents content interception |
| Spoofed device | User sees device name; must manually accept |
| Replay attack | TLS session binding; unique session IDs |
| File content privacy | TLS encryption; files not visible to observers |
| Certificate forgery | "Everyone" mode doesn't validate cert authority |

---

## 11. Linux Feasibility

### Summary: ✅ CONFIRMED — Primary Development Platform

| Capability | Status | Notes |
|-----------|--------|-------|
| AWDL (via OWL/filin-rs) | ✅ | Requires compatible Wi-Fi adapter |
| BLE advertising | ✅ | BlueZ / HCI direct |
| mDNS | ✅ | Avahi or manual implementation |
| AirDrop receive | ✅ | Validated on iOS 18 |
| AirDrop send | 🧪 | Less reliable |
| Monitor mode | ✅ | Well-supported with right hardware |
| Development tools | ✅ | Wireshark, tcpdump, hcitool, etc. |

Linux is the clear choice for the development/research platform.

---

## 12. Windows Feasibility

### Summary: 🧪 EXPERIMENTAL — Significant Challenges

| Capability | Status | Notes |
|-----------|--------|-------|
| BLE advertising | ✅ | WinRT `BluetoothLEAdvertisementPublisher` |
| BLE scanning | ✅ | WinRT `BluetoothLEAdvertisementWatcher` |
| mDNS | ✅ | Native support (Win10 1703+) |
| Wi-Fi Direct | ✅ | `Windows.Devices.WiFiDirect` |
| Monitor mode | 🔴 | Not supported on standard Windows drivers |
| Packet injection | 🔴 | Not available through standard APIs |
| AWDL | 🔴 | No implementation exists for Windows |
| Raw Wi-Fi access | 🔴 | Requires specialized drivers (e.g., Npcap) |

### Windows AirDrop Strategy Options

| Strategy | Feasibility | Complexity |
|----------|-------------|------------|
| **A: External USB adapter with Linux driver** | 🧪 EXPERIMENTAL | Very High — requires custom driver wrapper |
| **B: Linux VM/container with USB passthrough** | 🟡 PROBABLE | High — VM manages AWDL, Windows handles UI |
| **C: Wait for Wi-Fi Aware (DMA)** | 🟡 PROBABLE | Medium — depends on Apple's timeline |
| **D: Native protocol only (no AirDrop)** | ✅ Works | Low — but doesn't meet AirDrop goal |

**Recommended:** Start with Strategy D (native protocol) + Strategy B (Linux VM for AirDrop), monitor DMA developments for Strategy C.

---

## 13. Android Feasibility

### Summary: 🟡 PROBABLE — Google Has Done It; Third-Party Is Harder

| Capability | Status | Notes |
|-----------|--------|-------|
| BLE advertising | ✅ | `BLUETOOTH_ADVERTISE` permission (Android 12+) |
| BLE scanning | ✅ | `BLUETOOTH_SCAN` permission |
| Wi-Fi Aware (NAN) | ✅ | API level 26+, hardware dependent |
| Wi-Fi Direct | ✅ | Standard P2P API |
| mDNS | ✅ | `NsdManager` API |
| Monitor mode | 🔴 | Not available without root |
| Packet injection | 🔴 | Not available without root |
| AWDL (third-party) | 🔴 | Not feasible without chipset vendor support |
| AWDL (Google/OEM) | ✅ | Pixel 10+ has it natively via firmware |
| Foreground service | ✅ | Required for background operation |
| Scoped storage | ⚠️ | Must handle file access properly |
| Rust via NDK/JNI | ✅ | Supported via `android-ndk-rs` |

### Android AirDrop Strategy

| Strategy | Feasibility | Notes |
|----------|-------------|-------|
| **A: Leverage Google's Quick Share AirDrop interop** | 🔴 | Private API, not available to third-party apps |
| **B: Root + custom Wi-Fi driver** | 🧪 EXPERIMENTAL | Very limited audience |
| **C: Wi-Fi Aware (DMA path)** | 🟡 PROBABLE | Best long-term strategy |
| **D: Native UniversalDrop protocol** | ✅ Works | Primary path for Android ↔ Windows |

---

## 14. Hardware Requirements

### 14.1 Development Hardware (Recommended)

```
Development PC:
  - OS: Ubuntu 24.04 LTS or Kali Linux
  - Kernel: 6.8+ (mainline)
  
External Wi-Fi Adapter (for AWDL):
  - PRIMARY: Alfa AWUS036ACH (Realtek RTL8812AU)
    - Monitor mode: ✅
    - Packet injection: ✅
    - 5 GHz: ✅
    - Driver: rtl8812au (mainline in kernel 6.14+)
  
  - FALLBACK: TP-Link TL-WN722N v1 (Atheros AR9271)
    - Monitor mode: ✅
    - Packet injection: ✅
    - 5 GHz: ❌ (2.4 GHz only → channel 6 only)
    - Driver: ath9k_htc (in-kernel)
    - Note: Proven with OWL but limited to channel 6

Bluetooth:
  - Any USB Bluetooth 4.0+ adapter
  - Or internal Bluetooth (most laptops)
  
Test iPhone:
  - iPhone 12 or newer
  - iOS 17+ (ideally iOS 18)
  - AirDrop set to "Everyone for 10 minutes"
  
Test Android:
  - Pixel 8+ (for Wi-Fi Aware testing)
  - Samsung Galaxy S24+ (alternative)
  - Android 14+
  
Test Windows:
  - Windows 11 23H2+
  - Standard Wi-Fi (for native protocol)
  - Same Alfa adapter (for experimental AWDL)
```

### 14.2 Why This Hardware

| Choice | Reason |
|--------|--------|
| **Alfa AWUS036ACH** | Best combination of monitor mode, injection, dual-band, Linux support |
| **AR9271 fallback** | Proven with OWL; ultra-reliable but 2.4 GHz only |
| **Ubuntu 24.04** | LTS stability, excellent driver support, large community |
| **Kali Linux** | Pre-installed wireless tools, OWL in repos |
| **iPhone 12+** | Modern AWDL implementation; representative of target users |
| **Pixel 8+** | Wi-Fi Aware support, good development experience |

---

## 15. Native UniversalDrop Protocol

### 15.1 Transport Comparison

| Feature | QUIC | TCP+TLS | Raw TCP | UDP |
|---------|------|---------|---------|-----|
| **Encryption** | ✅ Built-in (TLS 1.3) | ✅ TLS overlay | ❌ Manual | ❌ Manual |
| **Reliability** | ✅ Stream-level | ✅ Connection-level | ✅ | ❌ Manual |
| **Multiplexing** | ✅ Native | ❌ Manual | ❌ | ❌ |
| **NAT Traversal** | 🟡 UDP-based (better) | 🔴 TCP (harder) | 🔴 | 🟡 |
| **Resumability** | ✅ 0-RTT reconnect | ❌ Manual | ❌ | ❌ |
| **LAN Performance** | ✅ Good | ✅ Good | ✅ Best | ✅ Good |
| **Android Support** | ✅ `quinn` via NDK | ✅ Native | ✅ | ✅ |
| **Windows Support** | ✅ `quinn` | ✅ Native | ✅ | ✅ |
| **Rust Libraries** | ✅ `quinn`, `s2n-quic` | ✅ `rustls`, `tokio` | ✅ `tokio` | ✅ `tokio` |
| **Debuggability** | 🟡 Newer tooling | ✅ Mature | ✅ Mature | 🟡 |
| **Complexity** | Medium-High | Medium | Low | Low |

### 15.2 Decision: TCP + TLS 1.3 (Primary), QUIC (Future)

**Rationale:**
- TCP+TLS is universally supported, well-debugged, and has mature Rust ecosystem
- QUIC adds complexity without significant LAN benefit (NAT traversal less relevant on LAN)
- TCP+TLS allows simpler initial implementation
- QUIC can be added as an alternative transport later
- File transfer is primarily sequential, reducing multiplexing benefit

### 15.3 Protocol Messages

```
┌──────────────────────────────────────────────────────┐
│                  Message Header                       │
├──────────┬──────────┬──────────┬──────────┬──────────┤
│ Magic(4) │ Version  │ MsgType  │ Flags(2) │ Length(4)│
│ "UDROP"  │ (1)      │ (1)      │          │          │
├──────────┴──────────┴──────────┴──────────┴──────────┤
│                  Payload (variable)                   │
├──────────────────────────────────────────────────────┤
│                  HMAC-SHA256 (32)                     │
└──────────────────────────────────────────────────────┘
```

### 15.4 Message Types

| ID | Message | Direction | Purpose |
|----|---------|-----------|---------|
| `0x01` | `HELLO` | Both | Initial handshake, version negotiation |
| `0x02` | `DEVICE_INFO` | Both | Exchange device capabilities |
| `0x03` | `PAIR_REQUEST` | Sender | Request pairing with receiver |
| `0x04` | `PAIR_RESPONSE` | Receiver | Accept/reject pairing |
| `0x05` | `AUTH` | Both | Authentication exchange |
| `0x10` | `TRANSFER_REQUEST` | Sender | Propose file transfer |
| `0x11` | `TRANSFER_ACCEPT` | Receiver | Accept transfer |
| `0x12` | `TRANSFER_REJECT` | Receiver | Reject transfer |
| `0x20` | `FILE_METADATA` | Sender | File name, size, hash, type |
| `0x21` | `FILE_CHUNK` | Sender | File data chunk |
| `0x22` | `FILE_ACK` | Receiver | Acknowledge chunk receipt |
| `0x23` | `FILE_RESUME` | Both | Resume from offset |
| `0x24` | `FILE_COMPLETE` | Sender | All chunks sent |
| `0x25` | `HASH_VERIFY` | Receiver | Full-file hash verification |
| `0x30` | `TRANSFER_COMPLETE` | Receiver | Transfer confirmed |
| `0xF0` | `CANCEL` | Both | Cancel operation |
| `0xFF` | `ERROR` | Both | Error with code and message |

### 15.5 Transfer Flow

```
Sender                              Receiver
  │                                      │
  │── HELLO ──────────────────────────► │
  │                      ◄── HELLO ─────│
  │                                      │
  │── DEVICE_INFO ────────────────────► │
  │                 ◄── DEVICE_INFO ────│
  │                                      │
  │── TRANSFER_REQUEST ───────────────► │
  │   {files: [{name, size, hash}]}      │
  │              ◄── TRANSFER_ACCEPT ───│
  │                                      │
  │── FILE_METADATA ──────────────────► │
  │   {name, size, sha256, mime_type}    │
  │                                      │
  │── FILE_CHUNK(0, data) ────────────► │
  │                    ◄── FILE_ACK(0) ─│
  │── FILE_CHUNK(1, data) ────────────► │
  │                    ◄── FILE_ACK(1) ─│
  │   ...                                │
  │── FILE_COMPLETE ──────────────────► │
  │                ◄── HASH_VERIFY ─────│
  │          ◄── TRANSFER_COMPLETE ─────│
```

### 15.6 Large File Transfer Design

```
File: 10 GB
Chunk size: 1 MB (configurable)
Total chunks: ~10,240

Stream → Chunk → Encrypt(TLS) → Transmit → ACK → Next

Resume Protocol:
  1. Connection lost at chunk 6,400 (6.25 GB)
  2. Reconnect
  3. Sender: FILE_RESUME {file_id, offset: 6,400, partial_hash}
  4. Receiver: FILE_RESUME_ACK {confirmed_offset: 6,400}
  5. Resume from chunk 6,400

Integrity:
  - Per-chunk: HMAC in message footer
  - Per-file: SHA-256 hash verified after all chunks received
  - Transfer: TRANSFER_COMPLETE only after hash verification passes
```

---

## 16. Security Threat Model

### 16.1 Threats and Mitigations

| ID | Threat | Impact | Mitigation | Residual Risk |
|----|--------|--------|------------|---------------|
| T1 | **Attacker on same Wi-Fi** | Intercept transfers | TLS encryption on all data | Low — TLS prevents content interception |
| T2 | **Malicious device spoofing** | Trick user into sending to wrong device | Device name shown to user; manual accept required | Medium — social engineering possible |
| T3 | **MITM attack** | Intercept/modify transfers | TLS with certificate pinning (native protocol); TOFU for AirDrop | Medium — first connection vulnerable in TOFU |
| T4 | **Replay attack** | Re-send previously captured files | TLS session binding; nonce in handshake | Low |
| T5 | **Malicious file received** | Malware execution | Files saved to sandbox; no auto-open; OS file scanning | Low — user must manually open |
| T6 | **Path traversal** | Write files outside designated directory | Sanitize all filenames; reject `..`; use UUID temp names | Low |
| T7 | **Resource exhaustion** | DoS via oversized transfers | Configurable size limits; rate limiting; timeout enforcement | Low |
| T8 | **Interrupted transfer** | Partial files left on disk | Atomic finalization (write to temp, rename on completion) | Low |
| T9 | **Key/credential leakage** | Identity compromise | Keys stored in OS credential store; never logged | Low |
| T10 | **BLE tracking** | Privacy violation | Rotating BLE MAC addresses; minimal advertisement data | Low |

### 16.2 Security Architecture

```
┌──────────────────────────────────────┐
│           Key Management              │
│  Ed25519 keypair per device           │
│  Stored in OS credential store        │
│  Never exported; never logged         │
├──────────────────────────────────────┤
│           TLS 1.3                     │
│  Self-signed certs (native protocol)  │
│  Certificate pinning after first pair │
│  TOFU (Trust On First Use) model      │
├──────────────────────────────────────┤
│           File Safety                 │
│  Filename sanitization                │
│  Path traversal prevention            │
│  Atomic writes (temp → rename)        │
│  Size limits                          │
│  Integrity verification (SHA-256)     │
├──────────────────────────────────────┤
│           Authorization               │
│  User must accept every transfer      │
│  Device pairing optional              │
│  Transfer notifications               │
└──────────────────────────────────────┘
```

---

## 17. Compatibility Matrix

### 17.1 Feature Feasibility

| Feature | Linux | Windows | Android |
|---------|-------|---------|---------|
| **BLE discovery** | ✅ Supported | ✅ Supported | ✅ Supported |
| **BLE advertising** | ✅ Supported | ✅ Supported | ✅ Supported (permission required) |
| **mDNS** | ✅ Supported | ✅ Supported | ✅ Supported |
| **AWDL** | ✅ Works (special HW) | 🔴 Not feasible (natively) | 🔴 Not feasible (third-party) |
| **AirDrop discovery** | ✅ Works | 🔴 Blocked by AWDL | 🔴 Blocked by AWDL |
| **AirDrop receive** | ✅ Confirmed (iOS 18) | 🔴 Blocked by AWDL | 🔴 Blocked by AWDL |
| **AirDrop send** | 🧪 Experimental | 🔴 Blocked by AWDL | 🔴 Blocked by AWDL |
| **iOS interop (receive from)** | ✅ Confirmed | ❓ Via bridge/VM | ❓ Via Wi-Fi Aware (future) |
| **iOS interop (send to)** | 🧪 Experimental | ❓ Unknown | ❓ Unknown |
| **Native UniversalDrop** | ✅ Supported | ✅ Supported | ✅ Supported |
| **Background receive** | ✅ Daemon/service | ✅ Windows service | ⚠️ Foreground service required |
| **Large files (10 GB+)** | ✅ Stream-based | ✅ Stream-based | ⚠️ Scoped storage limits |
| **Multiple files** | ✅ Supported | ✅ Supported | ✅ Supported |
| **Resume** | ✅ Supported | ✅ Supported | ✅ Supported |

### 17.2 Key Insight

> AWDL is the bottleneck for AirDrop on Android and Windows. The native UniversalDrop protocol works on all platforms without this constraint.

---

## 18. Recommended Architecture

```
                        UNIVERSALDROP
                             │
                ┌────────────┼────────────┐
                │                         │
                ▼                         ▼
        AIRDROP MODULE              NATIVE MODULE
                │                         │
        ┌───────┼───────┐         ┌───────┼───────┐
        │       │       │         │       │       │
        ▼       ▼       ▼         ▼       ▼       ▼
      BLE    AWDL   AirDrop    mDNS    TCP/TLS  Protocol
     Advrt   Link   HTTP/TLS   Disc    Transport Messages
        │       │       │         │       │       │
        └───────┼───────┘         └───────┼───────┘
                │                         │
                ▼                         ▼
        ┌──────────────────────────────────┐
        │         TRANSFER CORE            │
        │  Chunking, Resume, Integrity,    │
        │  File I/O, Progress, Cancel      │
        ├──────────────────────────────────┤
        │         SECURITY CORE            │
        │  Keys, Certs, TLS, Auth,         │
        │  File sanitization               │
        ├──────────────────────────────────┤
        │         PLATFORM LAYER           │
        │  OS-specific APIs, storage,      │
        │  notifications, permissions      │
        └──────────┬───────────────────────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
        ▼          ▼          ▼
     Android    Windows     Linux
      App        App       CLI/Daemon
```

### 18.1 Technology Stack Decision

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Core Language** | Rust | Memory safety, cross-platform, excellent networking, FFI to Android/Windows |
| **AirDrop Module** | Rust | Must interface directly with Wi-Fi/BLE hardware |
| **Native Protocol** | Rust (`tokio` + `rustls`) | Async networking, TLS, cross-platform |
| **Android App** | Kotlin + Rust (via JNI/NDK) | Native Android UI + Rust core logic |
| **Windows App** | Rust + WinRT | Or Rust core + C#/WinUI frontend |
| **Linux CLI** | Rust | Primary development/debugging tool |
| **Serialization** | `serde` + custom binary | Efficient, type-safe message handling |
| **Crypto** | `ring` or `rustcrypto` | Well-audited Rust crypto libraries |
| **TLS** | `rustls` | Pure-Rust TLS, no OpenSSL dependency |
| **mDNS** | `mdns-sd` crate or custom | Service discovery |
| **BLE** | `btleplug` (Rust) | Cross-platform BLE library |
| **Build** | Cargo workspaces | Monorepo with shared core |

---

## 19. License / Third-Party Audit

### 19.1 Critical License Analysis

| Project | License | Can We Copy Code? | Can We Link? | Strategy |
|---------|---------|-------------------|-------------|----------|
| **OpenDrop** | GPLv3 | 🔴 Only if entire project is GPLv3 | 🔴 Copyleft obligation | ✅ Study protocol; reimplement independently |
| **OWL** | GPLv3 | 🔴 Same as above | 🔴 Same | ✅ Study AWDL behavior; reimplement independently |
| **opendrop-rs** | GPLv3 | 🔴 Same as above | 🔴 Same | ✅ Study architecture; reimplement independently |
| **LocalSend** | MIT | ✅ Yes, with attribution | ✅ Yes | ✅ Reference protocol design |
| **rustls** | Apache-2.0/MIT | ✅ Yes | ✅ Yes | ✅ Use directly |
| **ring** | ISC-style | ✅ Yes | ✅ Yes | ✅ Use directly |
| **tokio** | MIT | ✅ Yes | ✅ Yes | ✅ Use directly |
| **quinn** (QUIC) | Apache-2.0/MIT | ✅ Yes | ✅ Yes | ✅ Use if QUIC adopted |
| **btleplug** | Apache-2.0/MIT | ✅ Yes | ✅ Yes | ✅ Use directly |

### 19.2 Recommended License Strategy

**Recommendation:** License UniversalDrop under **Apache-2.0 OR MIT** (dual license), which is the Rust ecosystem standard.

**Justification:**
- All core dependencies (rustls, tokio, ring, btleplug) are Apache-2.0/MIT compatible
- GPLv3 components (OpenDrop, OWL, opendrop-rs) are used as **protocol references only** — the implementation is independent
- Dual licensing maximizes compatibility for downstream users

**Requirements:**
- `THIRD_PARTY_NOTICES` file listing all dependencies and their licenses
- No code copied from GPLv3 projects — only protocol behavior studied
- Independent Rust implementation of AWDL and AirDrop protocol layers

---

## 20. Risk Register

| ID | Risk | Probability | Impact | Severity | Mitigation | Fallback |
|----|------|:-----------:|:------:|:--------:|------------|----------|
| R1 | AWDL requires special Wi-Fi hardware | Certain | High | **CRITICAL** | Document compatible hardware; recommend specific adapters | Native protocol for devices without compatible hardware |
| R2 | Windows cannot run AWDL natively | Very High | High | **CRITICAL** | Linux VM/bridge architecture | Windows-only native protocol; no AirDrop on Windows |
| R3 | Android AWDL not accessible to third-party | Very High | High | **CRITICAL** | Monitor DMA/Wi-Fi Aware developments | Native protocol; link to Google's Quick Share |
| R4 | Apple changes AirDrop protocol | Medium | High | **HIGH** | Version detection; modular protocol layer | Maintain compatibility matrix; rapid response |
| R5 | Apple-signed identity required for new features | High | Medium | **HIGH** | Focus on "Everyone" mode; document limitations | Clearly label as "Everyone mode only" |
| R6 | BLE advertising format changes | Medium | Medium | **HIGH** | Monitor Apple's BLE changes; version detection | Fall back to manual AWDL activation |
| R7 | Monitor mode driver instability | Medium | Medium | **MEDIUM** | Tested hardware list; fallback drivers | Specific hardware recommendations |
| R8 | Large file transfer stalls (half-duplex) | Medium | Medium | **MEDIUM** | Implement retry/resume; optimize chunk size | Reduce transfer speed; increase reliability |
| R9 | GPLv3 code accidentally included | Low | High | **MEDIUM** | Code review; license scanning tools | Remove and reimplement |
| R10 | Performance insufficient for production | Medium | Medium | **MEDIUM** | Benchmark early; optimize hot paths | Accept lower throughput vs. native AirDrop |
| R11 | iOS blocks non-Apple BLE advertisements | Low | Very High | **HIGH** | Test on multiple iOS versions | Rely on AWDL-only discovery (no BLE trigger) |
| R12 | Android vendor blocks custom BLE advertising | Medium | Medium | **MEDIUM** | Test on multiple vendors; use standard APIs | Limit to tested devices |
| R13 | Security vulnerability in protocol | Medium | Very High | **HIGH** | Security audit; fuzzing; peer review | Responsible disclosure; rapid patch |
| R14 | Maintenance complexity across 3 platforms | High | Medium | **MEDIUM** | Shared Rust core; platform-specific thin layers | Reduce platform count if resources limited |
