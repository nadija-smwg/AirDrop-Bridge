# UniversalDrop — Hardware Compatibility

**Date:** September 2026

---

## 1. Wi-Fi Adapter Requirements (AWDL)

| Requirement | Necessity | Verification Command |
|-------------|----------|---------------------|
| Monitor mode | 🔴 CRITICAL | `sudo airmon-ng start <iface>` |
| Packet injection | 🔴 CRITICAL | `sudo aireplay-ng --test <iface>` |
| 5 GHz support | ⚠️ IMPORTANT | `iw phy <phy> info \| grep -A20 "Frequencies"` |
| Channel switching | ⚠️ IMPORTANT | `sudo iw dev <iface> set channel 44` |

---

## 2. Recommended Adapters

### Tier 1 — Recommended

| Adapter | Chipset | Driver | Bands | Monitor | Injection | Price | Notes |
|---------|---------|--------|-------|---------|-----------|-------|-------|
| **Alfa AWUS036ACH** | RTL8812AU | `rtl8812au` | 2.4+5 GHz | ✅ | ✅ | ~$40 | Best overall; dual-band; widely tested |
| **Alfa AWUS036ACHM** | MT7612U | `mt76` | 2.4+5 GHz | ✅ | ✅ | ~$50 | In-kernel driver; excellent stability |

### Tier 2 — Budget / Backup

| Adapter | Chipset | Driver | Bands | Monitor | Injection | Price | Notes |
|---------|---------|--------|-------|---------|-----------|-------|-------|
| **TP-Link TL-WN722N v1** | AR9271 | `ath9k_htc` | 2.4 GHz | ✅ | ✅ | ~$20 | ⚠️ Must be v1 (not v2/v3); 2.4 GHz only → channel 6 only |
| **Panda PAU09** | RT5572 | `rt2800usb` | 2.4+5 GHz | ✅ | ✅ | ~$25 | Decent dual-band option |

### Tier 3 — NOT Compatible

| Chipset | Driver | Why |
|---------|--------|-----|
| Intel AX200/210 | `iwlwifi` | No monitor mode; no injection |
| Broadcom BCM43xx | `brcmfmac` | Extremely limited monitor/injection |
| Qualcomm QCA9377 | `ath10k` | Monitor mode partial; no injection |
| Realtek RTL8821CE | `rtw88` | No injection support |

---

## 3. Bluetooth Requirements

| Requirement | Specification | Notes |
|-------------|--------------|-------|
| Bluetooth LE (BLE) | 4.0+ | Required for AirDrop discovery |
| Advertising support | Must support custom manufacturer data | Most modern BT4.0+ adapters work |
| Linux driver | BlueZ 5.x | Standard in Ubuntu/Kali |

### Recommended Bluetooth Adapters

Most internal laptop Bluetooth works. If external adapter needed:

| Adapter | Chipset | BLE | Price | Notes |
|---------|---------|-----|-------|-------|
| **Any CSR/Cambridge Silicon Radio** | CSR8510 | ✅ | ~$10 | Ubiquitous; well-supported |
| **TP-Link UB500** | RTL8761B | ✅ | ~$15 | BT 5.0; good Linux support |

---

## 4. Development Machine Requirements

### Minimum
```
CPU:        Any modern x86_64 (Intel/AMD)
RAM:        8 GB
Storage:    50 GB free
USB:        USB 3.0 (for Wi-Fi adapter throughput)
OS:         Ubuntu 24.04 LTS or Kali Linux
Kernel:     6.1+ (6.8+ recommended)
```

### Recommended
```
CPU:        Intel i5/Ryzen 5 or better
RAM:        16 GB
Storage:    100 GB SSD
USB:        USB 3.0 (multiple ports)
Network:    Wired Ethernet (for internet while Wi-Fi in monitor mode)
OS:         Ubuntu 24.04 LTS
Kernel:     6.8+
```

### Important: NOT Virtual Machine
```
⚠️ AWDL development MUST use bare-metal Linux
⚠️ VMs (VirtualBox, VMware, WSL2) cannot passthrough USB Wi-Fi for monitor mode
⚠️ Exception: USB passthrough in VMware/QEMU MAY work but is unreliable
```

---

## 5. Test Device Requirements

### iPhone (Required)
| Model | iOS | AirDrop Mode | Purpose |
|-------|-----|-------------|---------|
| iPhone 12+ | 17+ | "Everyone for 10 min" | Primary test device |
| Multiple models | Various | Both modes | Compatibility testing |

### Android (For native protocol)
| Device | Android | Wi-Fi Aware | Purpose |
|--------|---------|-------------|---------|
| Pixel 8/8 Pro | 14+ | ✅ | Primary test device |
| Samsung Galaxy S24 | 14+ | ✅ | Alternative vendor |
| Budget device | 12+ | ❌ | Compatibility testing |

### Windows (For native protocol)
| OS | Version | Purpose |
|----|---------|---------|
| Windows 11 | 23H2+ | Primary test |
| Windows 10 | 22H2 | Compatibility |

---

## 6. Living Compatibility Matrix

This matrix should be updated as testing progresses:

| Platform | OS Version | Device | Wi-Fi Chipset | BT Chipset | AirDrop Recv | AirDrop Send | UD Recv | UD Send | Notes | Test Date |
|----------|-----------|--------|--------------|-----------|:------------:|:------------:|:-------:|:-------:|-------|-----------|
| Linux | Ubuntu 24.04 | PC + Alfa ACH | RTL8812AU | CSR8510 | ⏳ | ⏳ | ⏳ | ⏳ | | |
| Linux | Kali 2024.3 | PC + TP-Link | AR9271 | Internal | ⏳ | ⏳ | ⏳ | ⏳ | | |
| Android | 14 | Pixel 8 | Internal | Internal | N/A | N/A | ⏳ | ⏳ | | |
| Windows | 11 23H2 | Desktop | Internal | Internal | N/A | N/A | ⏳ | ⏳ | | |
| iPhone | iOS 18 | iPhone 14 | N/A | N/A | N/A | ⏳ | N/A | N/A | Sender only | |

Legend: ✅ Passed | ❌ Failed | ⏳ Not Yet Tested | N/A Not Applicable
