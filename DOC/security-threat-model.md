# UniversalDrop — Security Threat Model

**Date:** September 2026

---

## 1. Security Architecture Overview

```
┌────────────────────────────────────────────────────┐
│                   APPLICATION                       │
│  File handling, metadata processing, UI             │
├────────────────────────────────────────────────────┤
│                AUTHORIZATION                        │
│  User accept/decline, device pairing, ACLs          │
├────────────────────────────────────────────────────┤
│               AUTHENTICATION                        │
│  Device identity (Ed25519), TLS certs, TOFU         │
├────────────────────────────────────────────────────┤
│                 ENCRYPTION                          │
│  TLS 1.3 (native), TLS 1.2+ (AirDrop compat)       │
├────────────────────────────────────────────────────┤
│                  TRANSPORT                          │
│  TCP/HTTPS (AirDrop), TCP+TLS (native)              │
├────────────────────────────────────────────────────┤
│                  DISCOVERY                          │
│  BLE, mDNS — minimal information exposure           │
└────────────────────────────────────────────────────┘
```

---

## 2. Device Identity Architecture

### Native UniversalDrop Protocol

```
Device Identity:
  - Ed25519 keypair generated on first launch
  - Private key stored in OS credential store:
    - Linux: libsecret / GNOME Keyring
    - Android: Android Keystore (TEE-backed)
    - Windows: Windows Credential Manager (DPAPI)
  - Public key used to derive device fingerprint
  - Self-signed X.509 certificate for TLS

Pairing:
  - Trust-On-First-Use (TOFU) model
  - First connection: user verifies device visually
  - Certificate fingerprint pinned after first pair
  - Subsequent connections verify pinned cert

Key Lifecycle:
  - Generated: on first app launch
  - Stored: OS credential store (never exported)
  - Rotated: on user request only
  - Revoked: on device reset / app uninstall
```

### AirDrop Compatibility

```
AirDrop Identity (for receiver):
  - Self-signed RSA 2048-bit certificate
  - Generated per session or persisted
  - No Apple identity validation needed for "Everyone" mode
  - Cannot participate in "Contacts Only" without Apple certs
```

---

## 3. Threat Model

### 3.1 Threat Actors

| Actor | Capability | Motivation |
|-------|-----------|------------|
| **Passive Wi-Fi observer** | Can monitor Wi-Fi traffic | Surveillance, data theft |
| **Active MITM** | Can intercept and modify traffic | Data theft, file injection |
| **Malicious device on LAN** | Can participate in discovery | Spam, phishing, social engineering |
| **Malicious file sender** | Can craft malicious files | Malware delivery, exploitation |
| **Physical proximity attacker** | Can be near target device | BLE tracking, identity discovery |

### 3.2 Threats and Mitigations

| ID | Threat | Category | Impact | Probability | Severity | Mitigation | Residual Risk |
|----|--------|----------|--------|-------------|----------|------------|---------------|
| **T01** | Passive interception of file transfer | Confidentiality | HIGH | Medium | **HIGH** | All transfers encrypted via TLS 1.3 / 1.2+. No plaintext data on wire. | LOW — TLS prevents content sniffing |
| **T02** | MITM on first connection (TOFU) | Integrity | HIGH | Low | **HIGH** | Display device fingerprint for manual verification. Warn on cert change. | MEDIUM — First connection is vulnerable |
| **T03** | Spoofed device name | Integrity | MEDIUM | Medium | **MEDIUM** | Show device fingerprint alongside name. Paired devices are pinned. | MEDIUM — Social engineering possible |
| **T04** | Replay of captured transfer | Integrity | LOW | Low | **LOW** | TLS session binding with unique nonces. Replay would fail TLS handshake. | LOW |
| **T05** | Malicious file delivery | Integrity | HIGH | Medium | **HIGH** | Files saved to quarantine folder. No auto-execution. OS-level malware scanning. Notify user before opening. | MEDIUM — User must still open file |
| **T06** | Path traversal in filename | Integrity | CRITICAL | Medium | **CRITICAL** | Strip all path components. Reject `..`, `/`, `\`. Use UUID temp names. Atomic rename to sanitized name. | LOW — Multiple defense layers |
| **T07** | Resource exhaustion (DoS) | Availability | MEDIUM | Medium | **MEDIUM** | Maximum file size configurable. Rate limiting on connections. Timeout on all operations. Maximum concurrent transfers. | LOW |
| **T08** | BLE identity tracking | Privacy | MEDIUM | High | **MEDIUM** | Rotate BLE advertisement data. Minimize exposed identity information. Use random MAC where platform allows. | MEDIUM — BLE inherently leaks proximity |
| **T09** | Hash reversal of AirDrop identity | Privacy | MEDIUM | Medium | **MEDIUM** | UniversalDrop BLE doesn't use contact hashes. For AirDrop compat: document limitation, use empty hash. | LOW (for native), MEDIUM (for AirDrop) |
| **T10** | Unauthorized file receipt | Availability | LOW | Medium | **LOW** | User must explicitly accept every transfer. "Discoverable" mode can be disabled. Auto-timeout on discoverability. | LOW |
| **T11** | Key extraction from device | Confidentiality | HIGH | Low | **HIGH** | Keys in hardware-backed keystore (Android TEE, Windows DPAPI). Never logged. Never transmitted. | LOW — requires device compromise |
| **T12** | Oversized plist/CPIO (bomb) | Availability | MEDIUM | Low | **MEDIUM** | Limit plist recursion depth. Limit CPIO entry count and total size. Memory-mapped I/O with limits. | LOW |
| **T13** | Connection flood | Availability | MEDIUM | Medium | **MEDIUM** | Rate limit new connections. Maximum concurrent connections. Connection timeout. | LOW |
| **T14** | Partial file corruption | Integrity | MEDIUM | Low | **LOW** | Per-chunk HMAC verification. Full-file SHA-256 hash after transfer. Atomic finalization (temp → rename). | LOW |

---

## 4. Security Design Principles

### 4.1 Defense in Depth
```
Layer 1: Network encryption (TLS)
Layer 2: Device authentication (certificate pinning)
Layer 3: User authorization (accept/decline dialog)
Layer 4: File safety (sanitization, quarantine, no auto-execute)
Layer 5: Integrity verification (hashing)
```

### 4.2 Least Privilege
- App requests only necessary permissions
- Files written to designated directories only
- Network access limited to local network
- No cloud communication
- No analytics or telemetry

### 4.3 Secure by Default
- Discoverable mode OFF by default (user must enable)
- Auto-timeout on discoverability (configurable, default 10 min)
- All transfers require manual acceptance
- TLS always enabled (no plaintext mode)

---

## 5. File Safety

### 5.1 Filename Sanitization

```rust
fn sanitize_filename(raw: &str) -> String {
    // 1. Extract basename only (no directory components)
    // 2. Reject ".." sequences
    // 3. Replace/remove forbidden characters: / \ : * ? " < > |
    // 4. Limit length to 255 bytes
    // 5. Handle empty names (use UUID)
    // 6. Handle reserved names on Windows (CON, PRN, AUX, NUL, COM1-9, LPT1-9)
    // 7. Trim leading/trailing dots and spaces (Windows)
    // 8. Preserve file extension
}
```

### 5.2 File Storage

```
Incoming Transfer:
  1. Create temp file: <temp_dir>/<uuid>.tmp
  2. Write chunks to temp file
  3. Verify SHA-256 hash matches expected
  4. Sanitize filename
  5. Atomic rename: <temp_dir>/<uuid>.tmp → <save_dir>/<sanitized_name>
  6. If name collision: append counter (e.g., photo (2).jpg)
  
On failure:
  - Delete temp file
  - Log error
  - Notify user
```

### 5.3 Never Log
```
❌ Private keys
❌ TLS session secrets
❌ Raw file contents
❌ User passwords/PINs
❌ Full device identifiers of other users
❌ Contact information hashes
```

### 5.4 Always Log
```
✅ Transfer status (start, progress, complete, error)
✅ Connection events (connect, disconnect)
✅ Authentication events (cert verification result)
✅ Discovery events (device found, service resolved)
✅ Error details (category, message — no secrets)
✅ Performance metrics (throughput, latency)
```

---

## 6. Platform-Specific Security

### Android
```
- Scoped Storage enforcement (no arbitrary filesystem access)
- Android Keystore for key storage (hardware-backed TEE)
- BLUETOOTH_ADVERTISE, BLUETOOTH_SCAN, BLUETOOTH_CONNECT permissions
- NEARBY_WIFI_DEVICES permission (Android 13+)
- Foreground Service notification (visible to user)
- POST_NOTIFICATIONS permission
```

### Windows
```
- Windows Credential Manager (DPAPI) for key storage
- Windows Firewall configuration (inbound rule for UniversalDrop port)
- UAC elevation NOT required for normal operation
- Windows Service runs under limited service account
- AppContainer isolation if packaged as MSIX
```

### Linux
```
- libsecret / GNOME Keyring for key storage
- No root required for native protocol (only for AirDrop/AWDL)
- Systemd service with sandboxing (PrivateTmp, ProtectSystem)
- File capabilities instead of setuid where possible
```
