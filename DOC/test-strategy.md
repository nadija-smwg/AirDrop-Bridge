# UniversalDrop — Test Strategy

**Date:** September 2026

---

## 1. Testing Levels

```
┌─────────────────────────────────────────┐
│  Level 5: Compatibility Testing          │
│  Multiple devices, OS versions, HW       │
├─────────────────────────────────────────┤
│  Level 4: Real-Device Testing            │
│  iPhone ↔ Linux, Android, Windows        │
├─────────────────────────────────────────┤
│  Level 3: Integration Testing            │
│  Cross-platform transfers, E2E flows     │
├─────────────────────────────────────────┤
│  Level 2: Component Testing              │
│  Module-level: AWDL, BLE, AirDrop, etc.  │
├─────────────────────────────────────────┤
│  Level 1: Unit Testing                   │
│  Functions, parsers, serializers         │
└─────────────────────────────────────────┘
```

---

## 2. Unit Tests

### What to Test

| Module | Test Cases |
|--------|-----------|
| **Binary Plist Parser** | Valid plists, malformed plists, deeply nested plists, empty plists, large plists |
| **CPIO Parser** | Valid archives, corrupt headers, path traversal attempts, oversized entries, multi-file archives |
| **AWDL Frame Parser** | Valid action frames, corrupted TLVs, all TLV types, length field overflow |
| **AWDL Frame Generator** | Round-trip (parse→generate→parse), field correctness, byte alignment |
| **Protocol Messages** | Serialization round-trips for all message types, version negotiation, invalid message handling |
| **File Chunking** | Correct chunk boundaries, last chunk handling, empty file, 1-byte file, exact chunk-size file |
| **SHA-256 Hashing** | Known-value verification, streaming hash, partial hash for resume |
| **File Name Sanitizer** | Path traversal (`../`), null bytes, reserved names (Windows), unicode, empty names, long names |
| **Crypto** | Key generation, signing, verification, certificate generation, TLS config |
| **Resume State** | Offset tracking, partial hash verification, state persistence, corrupt state handling |
| **mDNS** | Service record generation, TXT record parsing, hostname resolution |

### Coverage Target
- Minimum 80% line coverage for core crates
- 100% coverage for security-critical paths (sanitization, crypto, parsing)

---

## 3. Integration Tests

### Linux ↔ Linux (Native Protocol)
```
Test: Two instances on same machine (loopback)
Cases:
  - [ ] Discovery via mDNS
  - [ ] Hello + Device Info exchange
  - [ ] Single file transfer (1 KB)
  - [ ] Single file transfer (100 MB)
  - [ ] Single file transfer (1 GB)
  - [ ] Multiple file transfer
  - [ ] Directory transfer
  - [ ] Transfer cancellation (sender)
  - [ ] Transfer cancellation (receiver)
  - [ ] Transfer rejection
  - [ ] Resume after disconnect
  - [ ] Resume after process restart
  - [ ] Integrity verification (intentional corruption)
  - [ ] Concurrent transfers
  - [ ] Timeout handling
```

### AirDrop Integration (Linux)
```
Test: linux-receiver with simulated AirDrop client
Cases:
  - [ ] /Discover request handling
  - [ ] /Ask request handling
  - [ ] /Upload handling (small file)
  - [ ] /Upload handling (large file)
  - [ ] Session continuity (/Ask → /Upload same TLS session)
  - [ ] TLS handshake
  - [ ] Binary plist round-trip
  - [ ] Invalid request handling
  - [ ] Connection timeout
```

---

## 4. Real-Device Test Matrix

### AirDrop Path (iPhone → Linux)

| iPhone Model | iOS Version | File Type | Size | Result | Date |
|-------------|-------------|-----------|------|--------|------|
| iPhone 12 | iOS 17.x | HEIC photo | ~3 MB | | |
| iPhone 12 | iOS 17.x | JPEG photo | ~5 MB | | |
| iPhone 12 | iOS 17.x | Video (MOV) | ~100 MB | | |
| iPhone 12 | iOS 17.x | PDF | ~2 MB | | |
| iPhone 12 | iOS 17.x | Multiple photos | ~20 MB | | |
| iPhone 13 | iOS 18.x | HEIC photo | ~3 MB | | |
| iPhone 14 | iOS 18.x | HEIC photo | ~3 MB | | |
| iPhone 15 | iOS 18.x | HEIC photo | ~3 MB | | |
| iPhone 16 | iOS 19.x | HEIC photo | ~3 MB | | |

### Native Protocol

| Sender | Receiver | File Type | Size | Result | Date |
|--------|----------|-----------|------|--------|------|
| Linux CLI | Linux CLI | Binary | 100 MB | | |
| Linux CLI | Linux CLI | Binary | 1 GB | | |
| Linux CLI | Linux CLI | Binary | 10 GB | | |
| Android app | Linux CLI | Photo | 5 MB | | |
| Android app | Windows app | Photo | 5 MB | | |
| Windows app | Android app | Document | 10 MB | | |
| Android app | Android app | Video | 500 MB | | |
| Windows app | Windows app | Archive | 1 GB | | |

### Cross-Platform Matrix

```
                 Receiver
              Linux  Android  Windows
Sender:
  Linux        ✅      □        □
  Android      □       □        □
  Windows      □       □        □
  iPhone*      □       □        □

* AirDrop path only
□ = Not yet tested
✅ = Tested and passing
```

---

## 5. Compatibility Testing

### Android Device Matrix

| Vendor | Model | Android Version | Wi-Fi Aware | BLE Adv | Result |
|--------|-------|----------------|-------------|---------|--------|
| Google | Pixel 8 | 14 | ✅ | ✅ | |
| Google | Pixel 8 | 15 | ✅ | ✅ | |
| Samsung | Galaxy S24 | 14 | ✅ | ✅ | |
| Samsung | Galaxy A54 | 14 | ❓ | ✅ | |
| OnePlus | 12 | 14 | ❓ | ✅ | |
| Xiaomi | 14 | 14 | ❓ | ✅ | |

### Windows Version Matrix

| Windows Version | Build | mDNS | BLE | Wi-Fi Direct | Result |
|----------------|-------|------|-----|-------------|--------|
| Windows 10 22H2 | 19045 | ✅ | ✅ | ✅ | |
| Windows 11 23H2 | 22631 | ✅ | ✅ | ✅ | |
| Windows 11 24H2 | 26100 | ✅ | ✅ | ✅ | |

### Wi-Fi Adapter Matrix (AWDL)

| Adapter | Chipset | Driver | Monitor | Injection | AWDL Sync | Transfer | Date |
|---------|---------|--------|---------|-----------|-----------|----------|------|
| Alfa AWUS036ACH | RTL8812AU | rtl8812au | | | | | |
| Alfa AWUS036ACHM | MT7612U | mt76 | | | | | |
| TP-Link TL-WN722N v1 | AR9271 | ath9k_htc | | | | | |

---

## 6. Performance Benchmarks

### Metrics to Track

| Metric | Target | How Measured |
|--------|--------|-------------|
| Discovery latency | < 3 seconds | Time from enable to device visible |
| Connection setup | < 1 second | Time from select to connected |
| Transfer throughput (LAN) | > 50 MB/s | Bytes transferred / time |
| Transfer throughput (AWDL) | > 5 MB/s | Limited by half-duplex |
| CPU usage (transfer) | < 25% | System monitor during 1 GB transfer |
| RAM usage (idle) | < 50 MB | Baseline memory consumption |
| RAM usage (transfer) | < 100 MB | Peak during large transfer |
| Resume latency | < 2 seconds | Time from reconnect to resumed transfer |

### Benchmark File Sizes
- 100 MB (typical photo collection)
- 1 GB (typical video)
- 5 GB (large project)
- 10 GB (stress test)
- 50 GB (stress test, if hardware permits)

---

## 7. Security Tests

### Fuzzing
- Protocol message parser (all message types)
- Binary plist parser
- CPIO parser
- File name sanitizer
- AWDL frame parser

### Penetration Tests
- Path traversal in filenames
- Oversized messages (DoS)
- Malformed TLS handshake
- Connection flood
- Replay attack
- Spoofed device name
- Truncated transfer
- Corrupt chunk data

### Automated Security Checks
- `cargo audit` — dependency vulnerability scan
- `clippy` — lint for unsafe code
- Memory sanitizers (ASAN, MSAN) on CI
- TLS configuration validation

---

## 8. Observability During Testing

### Structured Log Format
```
[2026-09-05T14:30:00Z] [INFO] [BLE] Advertisement broadcast started
[2026-09-05T14:30:01Z] [DEBUG] [AWDL] PSF received from master 00:11:22:33:44:55
[2026-09-05T14:30:01Z] [INFO] [AWDL] Synced to master, AW period: 16 TU
[2026-09-05T14:30:02Z] [INFO] [MDNS] Service _airdrop._tcp registered on port 8770
[2026-09-05T14:30:05Z] [INFO] [AIRDROP] /Discover request from "iPhone"
[2026-09-05T14:30:06Z] [INFO] [AIRDROP] /Ask request: IMG_4821.HEIC (8.4 MB)
[2026-09-05T14:30:06Z] [INFO] [TRANSFER] Transfer accepted by user
[2026-09-05T14:30:07Z] [INFO] [TRANSFER] Receiving: 12% (1.0 MB / 8.4 MB)
[2026-09-05T14:30:10Z] [INFO] [TRANSFER] Complete: IMG_4821.HEIC (SHA-256 verified)
[2026-09-05T14:30:10Z] [INFO] [SECURITY] File saved: /home/user/Downloads/UniversalDrop/IMG_4821.HEIC
```

### Debug Levels
```
ERROR  — Failures that prevent operation
WARN   — Unexpected but recoverable conditions
INFO   — Key events (discovery, transfer start/complete)
DEBUG  — Protocol details (frame contents, timing)
TRACE  — Raw data (hex dumps, packet captures)
```
