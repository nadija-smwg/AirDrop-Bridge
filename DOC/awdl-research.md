# UniversalDrop — AWDL Research

**Date:** September 2026
**Sources:** SEEMOO Lab (TU Darmstadt), OWL project, academic papers, opendrop-rs

---

## 1. What is AWDL?

Apple Wireless Direct Link (AWDL) is a proprietary peer-to-peer Wi-Fi protocol developed by Apple for AirDrop, AirPlay, and Handoff. It enables direct device-to-device communication without requiring a shared access point.

**Status:** 🧪 EXPERIMENTAL for non-Apple platforms

---

## 2. Architecture

```
┌────────────────────────────────────────────────────┐
│                 AWDL Protocol                       │
├────────────────────────────────────────────────────┤
│ Peer Discovery & Master Election                    │
│   - Master Indication Frames (MIF)                  │
│   - Master metric comparison                        │
│   - Dynamic role assignment                         │
├────────────────────────────────────────────────────┤
│ Synchronization                                     │
│   - Periodic Synchronization Frames (PSF)           │
│   - Clock sync to master                            │
│   - AW schedule distribution                        │
├────────────────────────────────────────────────────┤
│ Channel Management                                  │
│   - Social channels: 6 (2.4G), 44, 149 (5G)        │
│   - Availability Windows (AW)                       │
│   - Channel hopping between infra and social        │
│   - Time-sliced operation                           │
├────────────────────────────────────────────────────┤
│ Data Transport                                      │
│   - Unicast and multicast frames                    │
│   - IPv6 link-local addressing                      │
│   - Standard TCP/UDP over AWDL interface            │
├────────────────────────────────────────────────────┤
│ IEEE 802.11 (Physical)                              │
│   - Standard Wi-Fi radio                            │
│   - Monitor mode for frame capture/injection        │
│   - Action frames for management                    │
└────────────────────────────────────────────────────┘
```

---

## 3. Channel Sequencing

AWDL uses a time-division approach to share the Wi-Fi radio between infrastructure (normal Wi-Fi) and peer-to-peer communication:

```
Timeline:
─────┬──────────┬──────┬──────────┬──────┬──────────
     │  Infra   │  AW  │  Infra   │  AW  │  Infra
     │  Ch 36   │ Ch 6 │  Ch 36   │ Ch 44│  Ch 36
─────┴──────────┴──────┴──────────┴──────┴──────────
     ← 16 TU →  ← AW → ← 16 TU →  ← AW →

TU = Time Unit = 1024 μs
AW = Availability Window
Infra = Infrastructure channel (normal Wi-Fi)
```

### Social Channels
| Channel | Band | Usage |
|---------|------|-------|
| 6 | 2.4 GHz | Primary social channel (always available) |
| 44 | 5 GHz | Additional social channel |
| 149 | 5 GHz | Additional social channel |

### Implications for Implementation
- Wi-Fi adapter must support rapid channel switching
- During AW, all AWDL peers are on the same social channel
- Outside AW, the device can return to its infrastructure channel
- This is why AirDrop can work while connected to Wi-Fi

---

## 4. Master Election

### Algorithm
1. Each device broadcasts Master Indication Frames (MIF)
2. MIF contains a "Master Metric" (priority value)
3. The device with the **highest** metric becomes the master
4. Master metric factors: hardware capabilities, software version, battery level
5. If current master leaves, re-election occurs automatically

### Master's Role
- Broadcasts Periodic Synchronization Frames (PSF)
- All other devices sync their clocks to the master
- Master determines AW schedule (when to tune to social channel)

### For UniversalDrop
- When joining an existing AWDL cluster (e.g., iPhone is master), synchronize to the iPhone's timing
- When no cluster exists, can become master with a low metric (defer to Apple devices)
- Priority: sync to existing cluster > become master

---

## 5. Synchronization

### Periodic Synchronization Frames (PSF)
```
PSF Contents:
  - Timestamp (master's clock)
  - AW sequence parameters
  - Channel sequence
  - Next AW timing
  - Extension periods
```

### Sync Process
1. Monitor Wi-Fi for PSF from master
2. Extract timing parameters
3. Adjust local clock to match master
4. Schedule AW openings to align with master's schedule
5. Periodically re-sync to prevent drift

### Timing Precision
- AW alignment must be accurate to ~100 μs
- Clock drift must be compensated
- Missed AWs result in failed communication

---

## 6. Frame Structures

### AWDL Action Frame (Management)
```
┌──────────────────────────────────────┐
│ IEEE 802.11 Header                    │
│   Frame Control: Action (0x00D0)      │
│   Duration: varies                    │
│   Addresses: DA, SA, BSSID            │
├──────────────────────────────────────┤
│ AWDL Vendor-Specific                  │
│   Category: Vendor Specific (127)     │
│   OUI: 00:17:F2 (Apple)              │
│   Type: AWDL (0x08)                  │
│   Version: varies                     │
├──────────────────────────────────────┤
│ AWDL TLVs (Type-Length-Value)         │
│   - Synchronization Parameters        │
│   - Election Parameters               │
│   - Channel Sequence                  │
│   - Service Parameters                │
│   - Data Path State                   │
└──────────────────────────────────────┘
```

### AWDL Data Frame
```
┌──────────────────────────────────────┐
│ IEEE 802.11 Header                    │
│   Frame Control: Data (0x0008)        │
│   Addresses: DA, SA, BSSID            │
│   Sequence Control                    │
├──────────────────────────────────────┤
│ LLC/SNAP Header                       │
│   EtherType: IPv6 (0x86DD)           │
├──────────────────────────────────────┤
│ IPv6 Payload                          │
│   Link-local address (fe80::...)     │
│   TCP/UDP payload                     │
└──────────────────────────────────────┘
```

---

## 7. Hardware Compatibility

### Requirements

| Requirement | Necessity | Notes |
|-------------|----------|-------|
| Monitor mode | 🔴 CRITICAL | Must capture/inject raw 802.11 frames |
| Packet injection | 🔴 CRITICAL | Must send custom action/data frames |
| 5 GHz support | ⚠️ IMPORTANT | Required for social channels 44, 149 |
| Channel switching | ⚠️ IMPORTANT | Must hop between infra and social channels |
| Concurrent interfaces | 🟢 NICE-TO-HAVE | Allows simultaneous monitor + managed mode |

### Chipset Compatibility Matrix

| Chipset | Linux Driver | Monitor | Injection | 5 GHz | Dual-Band | AWDL Tested | Recommended |
|---------|-------------|---------|-----------|-------|-----------|-------------|-------------|
| Atheros AR9271 | `ath9k_htc` | ✅ | ✅ | ❌ | ❌ | ✅ (OWL) | 🟡 Budget option (ch 6 only) |
| MediaTek MT7612U | `mt76` | ✅ | ✅ | ✅ | ✅ | 🧪 Likely | ✅ Best value |
| Realtek RTL8812AU | `rtl8812au` | ✅ | ✅ | ✅ | ✅ | 🧪 Reports | ✅ Widely available |
| Realtek RTL8814AU | `rtl8814au` | ✅ | ✅ | ✅ | ✅ | ❓ | 🟡 High-end |
| Intel AX200 | `iwlwifi` | ❌ | ❌ | ✅ | ✅ | ❌ | 🔴 Incompatible |
| Intel AX210 | `iwlwifi` | ❌ | ❌ | ✅ | ✅ | ❌ | 🔴 Incompatible |
| Broadcom BCM43xx | `brcmfmac` | ❌ mostly | ❌ | varies | varies | ❌ | 🔴 Incompatible |
| Qualcomm Atheros QCA9377 | `ath10k` | 🟡 | ❌ | ✅ | ✅ | ❌ | 🔴 No injection |

### Recommended USB Adapters

| Adapter | Chipset | Price Range | Notes |
|---------|---------|-------------|-------|
| **Alfa AWUS036ACH** | RTL8812AU | $30-50 | Dual-band, high power, widely used |
| **Alfa AWUS036ACHM** | MT7612U | $40-60 | MediaTek, excellent kernel support |
| **TP-Link TL-WN722N v1** | AR9271 | $15-25 | Budget; 2.4 GHz only; ensure v1 (not v2/v3) |
| **Panda PAU09** | RT5572 | $20-35 | Dual-band, decent injection support |

---

## 8. Platform Feasibility

### Linux ✅ CONFIRMED
```
Mechanism: OWL / filin-rs userspace implementation
Requirements:
  - Compatible USB Wi-Fi adapter (monitor mode + injection)
  - Root access (for raw sockets, monitor mode)
  - Linux kernel 5.4+ (4.x works but older drivers)
  - libpcap for frame capture/injection
  - Netlink API for interface management

Status: Working. Both OWL (C) and filin-rs (Rust) have validated implementations.
```

### Windows 🔴 NOT CURRENTLY FEASIBLE (natively)
```
Blockers:
  - Windows Wi-Fi drivers do not expose monitor mode
  - No packet injection API in standard Windows
  - Npcap provides some capture but not full injection on most chipsets
  - No userspace Wi-Fi driver support comparable to Linux

Workarounds:
  - External USB adapter with custom Windows driver (experimental, unreliable)
  - Linux VM with USB passthrough (functional but requires VM setup)
  - Windows Subsystem for Linux 2 (WSL2) — no USB passthrough for Wi-Fi

Realistic Path: Linux VM bridge or wait for Wi-Fi Aware (DMA)
```

### Android 🔴 NOT FEASIBLE (third-party)
```
Blockers:
  - No monitor mode access from userspace (even with root on most devices)
  - Wi-Fi chipset firmware locked by vendors
  - No raw 802.11 frame injection API
  - Google's Pixel 10 AirDrop uses custom firmware-level AWDL (not accessible to apps)

Exception: Rooted devices with specific chipsets MAY support monitor mode
  - Nexus 5/6 (historical)
  - Some MediaTek devices
  - Very limited audience

Realistic Path: Wi-Fi Aware (DMA-mandated) or relay bridge via Linux
```

---

## 9. Implementation Strategy for UniversalDrop

### Phase 1: Research (Linux)
1. Capture AWDL frames from iPhone using Wireshark on monitor interface
2. Parse and document frame formats
3. Identify channel sequence and timing from captured PSFs
4. Validate against academic papers

### Phase 2: Implementation (Rust, Linux)
1. Implement AWDL frame parser (action frames, TLVs)
2. Implement frame generator
3. Implement channel hopping (tune to social channel on schedule)
4. Implement sync (parse PSF, align local clock)
5. Implement election (participate with low priority)
6. Create virtual AWDL interface (via tun/tap or Netlink)
7. Bridge IPv6 traffic over AWDL

### Phase 3: Validation
1. Test sync to iPhone's AWDL cluster
2. Verify bidirectional frame exchange
3. Verify IPv6 connectivity over AWDL
4. Test mDNS service discovery over AWDL interface

### Key Design Decisions
- **Independent Rust implementation** (not forked from OWL/GPLv3)
- Protocol behavior validated against captured packets
- Timing parameters tuned through empirical testing
- Error handling for timing drift, missed AWs, master changes
