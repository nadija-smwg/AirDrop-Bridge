# UniversalDrop — AirDrop Protocol Research

**Date:** September 2026
**Sources:** SEEMOO Lab (TU Darmstadt), CISPA, IEEE WOOT 2026, opendrop-rs, hexway research

---

## 1. Protocol Stack Overview

AirDrop operates through a 7-layer stack:

```
Layer 7: Application    — File transfer logic, UI integration
Layer 6: AirDrop HTTP   — /Discover, /Ask, /Upload over HTTPS
Layer 5: TLS            — Encryption, optional identity verification
Layer 4: TCP            — Reliable transport
Layer 3: IPv6           — Link-local addressing over AWDL
Layer 2: AWDL           — Peer-to-peer Wi-Fi link layer
Layer 1: BLE + Wi-Fi    — Physical discovery and transport
```

---

## 2. BLE Discovery Layer

### Purpose
Low-power proximity paging to initiate the AirDrop process.

### Transport
Bluetooth Low Energy (BLE) advertisements

### Format
```
┌──────────────┬──────────────┬──────────────────────┐
│ AD Length     │ AD Type 0xFF │ Company ID: 0x4C 0x00│
│ (1 byte)     │ (1 byte)     │ (2 bytes, Apple Inc) │
├──────────────┴──────────────┴──────────────────────┤
│ Type: 0x05 (AirDrop)                               │
├────────────────────────────────────────────────────┤
│ Flags (1 byte)                                      │
├────────────────────────────────────────────────────┤
│ Short Identity Hash (variable, ~2 bytes truncated)  │
│ Derived from SHA-256 of contact email/phone         │
├────────────────────────────────────────────────────┤
│ Apple ID Hash (variable)                            │
├────────────────────────────────────────────────────┤
│ AWDL State Information                              │
└────────────────────────────────────────────────────┘
```

### Security Implications
- Short identity hash is truncatable → brute-force reversal is possible (hexway research)
- Rotating MAC addresses mitigate tracking but do not prevent hash reversal
- 🧪 EXPERIMENTAL: Whether current iOS validates BLE payload beyond Company ID

### Required APIs
| Platform | API | Notes |
|----------|-----|-------|
| Linux | BlueZ HCI, `hcitool` | Full control over advertisement payload |
| Android | `BluetoothLeAdvertiser.startAdvertising()` | Can set manufacturer data; some vendor restrictions |
| Windows | `BluetoothLEAdvertisementPublisher` | Can set manufacturer-specific data |

### Implementation Difficulty: **Medium**
The format is well-understood. The challenge is ensuring iOS actually responds to non-Apple BLE sources.

---

## 3. AWDL Layer

### Purpose
Establish a direct peer-to-peer Wi-Fi link between devices without requiring a shared access point.

### Transport
IEEE 802.11 action frames and data frames on "social channels"

### Channel Architecture
```
Infrastructure Wi-Fi: Channel X (e.g., 36)
    │
    │ ← Device normally operates here
    │
    │ ──── AWDL Availability Window ────
    │      │
    │      ▼
Social Channel: 6, 44, or 149
    │      │
    │      │ ← AWDL traffic here
    │      │
    │ ──── End of AW ────
    │
    ▼
Infrastructure Wi-Fi: Channel X
    │
    │ ← Back to normal operation
```

### Frame Types

#### Master Indication Frame (MIF)
```
Purpose: Announce master election candidacy
Fields:
  - Master Address (6 bytes)
  - Master Metric (4 bytes) — higher = more likely to be elected
  - Counter
  - Synchronization tree depth
```

#### Periodic Synchronization Frame (PSF)
```
Purpose: Clock synchronization within the cluster
Fields:
  - Timing parameters
  - Availability window schedule
  - Channel sequence
  - Extension parameters
```

#### Availability Window Parameters
```
Purpose: Define when device is available on social channel
Fields:
  - AW period (typically 16 TU = 16384 μs)
  - AW duration
  - AW offset
  - Channel number
  - Guard time
```

### Required Interfaces
- Monitor mode Wi-Fi interface
- Frame injection capability
- Channel switching capability

### Required Privileges
- Root / sudo (Linux)
- Raw socket access
- Monitor mode activation

### Implementation Difficulty: **Very High**
This is the most challenging layer. Timing-critical, hardware-dependent, and requires deep wireless driver integration.

---

## 4. mDNS / Bonjour Layer

### Purpose
Advertise and discover the `_airdrop._tcp` service over the AWDL interface.

### Transport
Multicast DNS (mDNS) over UDP port 5353 on the AWDL interface

### Service Format
```
Service: _airdrop._tcp.local.
Port: <dynamic, e.g., 8770>
Host: <hostname>.local.
TXT Records:
  - flags=<capability flags>
```

### Required Interfaces
- AWDL virtual network interface (from Layer 2)
- IPv6 link-local address on AWDL interface

### Implementation Difficulty: **Low**
mDNS is well-documented and supported. The only complexity is binding to the correct (AWDL) interface.

---

## 5. AirDrop HTTP/TLS Layer

### Purpose
Application-level file transfer protocol over HTTPS.

### Transport
HTTPS (HTTP/1.1 over TLS 1.2+) on the port advertised via mDNS

### TLS Configuration
```
Protocol: TLS 1.2+ (TLS 1.3 preferred)
Certificate: Self-signed RSA 2048-bit (for "Everyone" mode)
Client Auth: Not required for "Everyone" mode
Session: Must persist across /Discover → /Ask → /Upload
Cipher suites: Standard (e.g., TLS_AES_256_GCM_SHA384)
```

### Endpoints

#### POST /Discover
```
Direction: Sender → Receiver
Purpose: Announce sender presence; get receiver info

Request (binary plist):
{
    "SenderRecordData": <binary>,        // Apple identity (may be empty for "Everyone")
    "SenderComputerName": "iPhone",      // Display name
    "SenderModelName": "iPhone15,2",     // Model identifier
    "SenderID": <UUID string>,           // Session UUID
    "BundleID": "com.apple.UIKit.activity.AirDrop"
}

Response (binary plist):
{
    "ReceiverComputerName": "UniversalDrop",
    "ReceiverModelName": "MacBookPro18,1",  // Spoof as Mac for compatibility
    "ReceiverMediaCapabilities": {
        "supportedFormats": [...]
    }
}
```

#### POST /Ask
```
Direction: Sender → Receiver
Purpose: Request permission to send specific files
Constraint: MUST be on same TLS session as /Discover

Request (binary plist):
{
    "SenderComputerName": "iPhone",
    "SenderModelName": "iPhone15,2",
    "SenderID": <UUID>,
    "Files": [
        {
            "FileName": "IMG_4821.HEIC",
            "FileType": "public.heic",
            "FileBomPath": "./IMG_4821.HEIC",
            "FileIsDirectory": false,
            "ConvertMediaFormats": false
        }
    ],
    "Icon": <thumbnail PNG data>,
    "SenderRecordData": <binary>
}

Response (binary plist):
// Accept:
{}  // Empty plist with HTTP 200

// Reject:
HTTP 401 or connection close
```

#### POST /Upload
```
Direction: Sender → Receiver
Purpose: Transfer file data
Constraint: MUST be on same TLS session as /Ask

Request body: CPIO archive
  Content-Type: application/x-cpio
  Optional: DVZip compression applied
  
  Archive contains:
    - File(s) as listed in /Ask
    - Metadata files (optional)

Response:
HTTP 200 with empty body (success)
```

### File Payload Format
```
CPIO Archive Structure:
├── filename1.ext          # Actual file data
├── filename2.ext          # Second file (if multiple)
└── TRAILER!!!             # CPIO end marker

DVZip: Adaptive compression (zlib-based)
  - Applied to archive before sending
  - Not always used (depends on file type)
```

### Important Protocol Constraints
1. **Same TLS Session:** /Ask and /Upload MUST use the same TCP connection and TLS session
2. **Session Timeout:** If too much time passes between /Ask and /Upload, iOS may timeout
3. **Binary Plist:** All structured data uses Apple's binary property list format, NOT XML plist
4. **Model Spoofing:** Receiver should identify as a Mac model for maximum compatibility
5. **Content-Type:** Request bodies use binary plist content type for /Discover and /Ask

### Implementation Difficulty: **High**
The protocol itself is conceptually simple (3 HTTP endpoints), but the details matter:
- Binary plist parsing/generation must be exact
- TLS session management is critical
- CPIO archive parsing must handle edge cases
- Response format must match iOS expectations exactly

---

## 6. Protocol State Machine

```
IDLE
  │
  │ BLE advertisement detected
  ▼
BLE_DISCOVERED
  │
  │ AWDL sync established
  ▼
AWDL_CONNECTED
  │
  │ mDNS service resolved
  ▼
SERVICE_DISCOVERED
  │
  │ TLS connection established
  ▼
TLS_CONNECTED
  │
  │ POST /Discover received
  ▼
DISCOVERED
  │
  │ POST /Ask received
  ▼
ASK_PENDING
  │
  │ User accepts
  ▼
ASK_ACCEPTED
  │
  │ POST /Upload received
  ▼
UPLOADING
  │
  │ Upload complete
  ▼
TRANSFER_COMPLETE
  │
  │ Connection closed
  ▼
IDLE
```

---

## 7. Known Vulnerabilities (CISPA AirFuzz 2026)

For awareness and defensive implementation:

| ID | Vulnerability | Layer | Impact |
|----|--------------|-------|--------|
| V1 | Binary plist recursion DoS | Application | CPU exhaustion |
| V2 | HTTP path router bypass | HTTP | Unexpected behavior |
| V3 | Pre-auth processing of untrusted data | HTTP/TLS | Information leak |

**Mitigation for UniversalDrop:**
- Limit plist recursion depth
- Validate all HTTP paths strictly
- Minimize pre-authentication processing
- Set timeouts on all operations
