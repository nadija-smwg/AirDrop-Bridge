# UniversalDrop — License Audit

**Date:** September 2026

---

## 1. Project License Decision

**UniversalDrop License:** Apache-2.0 OR MIT (dual license)

This is the standard Rust ecosystem licensing convention. It allows:
- Commercial use
- Modification
- Distribution
- Patent use (Apache-2.0)
- Private use
- No copyleft obligation

---

## 2. External Project License Analysis

### 2.1 Reference Projects (Protocol Study Only — NO Code Reuse)

| Project | License | Can Copy Code? | Can Link/Depend? | Strategy |
|---------|---------|:--------------:|:----------------:|----------|
| seemoo-lab/opendrop | **GPLv3** | 🔴 NO | 🔴 NO (copyleft) | Study protocol behavior only; independent implementation |
| seemoo-lab/owl | **GPLv3** | 🔴 NO | 🔴 NO (copyleft) | Study AWDL behavior only; independent implementation |
| ayourtch-llm/opendrop-rs | **GPLv3** | 🔴 NO | 🔴 NO (copyleft) | Study architecture only; independent implementation |

#### GPLv3 Implications
- Any code copied from GPLv3 projects would require UniversalDrop to be GPLv3
- Linking (statically or dynamically) to GPLv3 libraries creates copyleft obligation
- Studying the protocol behavior and reimplementing independently is permissible
- The protocol itself (AirDrop, AWDL) is not copyrightable; implementations are

#### Safeguards
1. No developer should view GPLv3 source code while writing the same module
2. Implementation should be derived from:
   - Academic papers (freely available)
   - Protocol specifications from packet captures
   - Official Apple documentation (where available)
   - Independent reverse engineering
3. Code review should verify no substantial similarity to GPLv3 implementations

### 2.2 Direct Dependencies (Planned)

| Crate | License | Compatible? | Notes |
|-------|---------|:-----------:|-------|
| `tokio` | MIT | ✅ | Async runtime |
| `rustls` | Apache-2.0 / MIT | ✅ | TLS implementation |
| `ring` | ISC-style (permissive) | ✅ | Crypto primitives |
| `serde` | Apache-2.0 / MIT | ✅ | Serialization |
| `serde_json` | Apache-2.0 / MIT | ✅ | JSON serialization |
| `plist` | MIT | ✅ | Apple property list parsing |
| `btleplug` | Apache-2.0 / MIT | ✅ | Cross-platform BLE |
| `pcap` | Apache-2.0 / MIT | ✅ | Packet capture (wraps libpcap) |
| `tracing` | MIT | ✅ | Structured logging |
| `clap` | Apache-2.0 / MIT | ✅ | CLI argument parsing |
| `sha2` | Apache-2.0 / MIT | ✅ | SHA-256 hashing |
| `ed25519-dalek` | BSD-3-Clause | ✅ | Ed25519 signatures |
| `x509-cert` | Apache-2.0 / MIT | ✅ | Certificate handling |
| `mdns-sd` | Apache-2.0 / MIT | ✅ | mDNS service discovery |
| `quinn` | Apache-2.0 / MIT | ✅ | QUIC (future) |
| `egui` | Apache-2.0 / MIT | ✅ | GUI (Windows) |
| `jni` | Apache-2.0 / MIT | ✅ | Java Native Interface (Android) |

### 2.3 System Dependencies

| Library | License | Linking | Notes |
|---------|---------|---------|-------|
| `libpcap` | BSD | Dynamic | System library; BSD-compatible |
| `BlueZ` | GPL-2.0 | System service | Accessed via D-Bus IPC (not linked); no copyleft obligation |
| `OpenSSL` | Apache-2.0 (3.x) | **NOT USED** | Using `rustls` instead to avoid OpenSSL licensing complexity |

---

## 3. Third-Party Notices Template

```
THIRD-PARTY NOTICES
====================

UniversalDrop includes or depends on the following third-party software:

---

tokio
License: MIT
Copyright (c) Tokio Contributors
https://github.com/tokio-rs/tokio

---

rustls
License: Apache-2.0 OR MIT
Copyright (c) rustls Contributors
https://github.com/rustls/rustls

---

ring
License: ISC-style
Copyright (c) Brian Smith
https://github.com/briansmith/ring

---

[... additional entries for each dependency ...]
```

---

## 4. Patent Considerations

### Apple Patents
- AWDL and AirDrop are covered by various Apple patents
- Interoperability implementations may have patent risk
- **Mitigation:** UniversalDrop is for research and personal use
- **Note:** EU DMA mandates interoperability, which may provide legal protection in EU jurisdictions
- Consult legal counsel before commercial distribution

### Protocol Patents
- Some Wi-Fi protocol extensions may be patent-encumbered
- Standard Wi-Fi (802.11) is covered by RAND (Reasonable and Non-Discriminatory) licensing
- AWDL-specific mechanisms may not have RAND coverage

---

## 5. Recommendations

1. **Use Apache-2.0 OR MIT dual license** for all UniversalDrop code
2. **Never copy code** from GPLv3 projects (OpenDrop, OWL, opendrop-rs)
3. **Use `rustls`** instead of OpenSSL to avoid licensing complexity
4. **Maintain THIRD_PARTY_NOTICES** updated with every dependency change
5. **Run `cargo deny`** in CI to automatically check dependency licenses
6. **Document all protocol references** (academic papers, captures, not GPLv3 source)
7. **Consult legal counsel** before commercial distribution regarding patent risk
8. **Add license headers** to all source files
