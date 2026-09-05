# UniversalDrop — Windows Feasibility Report

**Date:** September 2026

---

## 1. Summary

| Aspect | Status |
|--------|--------|
| Native UniversalDrop protocol | ✅ Fully feasible |
| BLE advertising | ✅ Supported via WinRT |
| mDNS | ✅ Native support (Win10 1703+) |
| Wi-Fi Direct | ✅ Supported via WinRT |
| AWDL / AirDrop | 🔴 NOT FEASIBLE natively |
| AirDrop via Linux VM bridge | 🟡 PROBABLE |

---

## 2. Available APIs

### BLE (Bluetooth Low Energy)

| API | Namespace | Purpose |
|-----|-----------|---------|
| `BluetoothLEAdvertisementPublisher` | `Windows.Devices.Bluetooth.Advertisement` | Broadcast BLE advertisements |
| `BluetoothLEAdvertisementWatcher` | `Windows.Devices.Bluetooth.Advertisement` | Scan for BLE advertisements |
| `BluetoothLEDevice` | `Windows.Devices.Bluetooth` | Connect to BLE devices |

**Status:** ✅ Full BLE advertising and scanning supported
**Access:** Via WinRT APIs (accessible from Rust via `windows-rs` crate)

### mDNS

| Feature | Support | Notes |
|---------|---------|-------|
| DNS-SD resolution | ✅ | Native in Win10 1703+ |
| `.local` hostname resolution | ✅ | Via DNS client service |
| Service registration | ⚠️ | May need manual registry config |

**Registry (if needed):**
```
HKLM\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient
  mDNSEnabled = 1 (DWORD)
```

### Wi-Fi Direct

| API | Namespace | Purpose |
|-----|-----------|---------|
| `WiFiDirectDevice` | `Windows.Devices.WiFiDirect` | P2P connections |
| `WiFiDirectAdvertisementPublisher` | Same | Advertise P2P availability |

**Note:** `WiFiDirect.Services` namespace is **deprecated**.

### Networking

| API | Purpose |
|-----|---------|
| WinSock2 | TCP/UDP sockets |
| SChannel / WinRT SSL | TLS connections |
| `windows-rs` | Rust bindings to Windows APIs |

---

## 3. AWDL on Windows: 🔴 NOT FEASIBLE (natively)

### Blockers

| Blocker | Explanation |
|---------|-------------|
| No monitor mode | Windows Wi-Fi drivers do not support monitor mode via public APIs |
| No packet injection | Standard Windows APIs cannot inject raw 802.11 frames |
| No AWDL implementation | No userspace or kernel AWDL stack exists for Windows |
| Npcap limitations | Npcap provides packet capture but injection is chipset-dependent and unreliable for AWDL |
| Driver model | Windows NDIS driver model doesn't expose low-level Wi-Fi management |

### Alternative Strategies for AirDrop on Windows

| Strategy | Feasibility | Complexity | User Experience |
|----------|-------------|------------|-----------------|
| **A: Linux VM + USB passthrough** | 🟡 PROBABLE | High | Must set up VM; USB adapter required |
| **B: WSL2 + networking** | 🔴 NOT FEASIBLE | N/A | WSL2 cannot access USB Wi-Fi hardware |
| **C: External USB adapter + custom Windows driver** | 🧪 EXPERIMENTAL | Very High | Requires driver development |
| **D: Wait for Wi-Fi Aware (DMA)** | 🟡 PROBABLE | Medium | Best UX when available |
| **E: Raspberry Pi bridge (headless)** | 🟡 PROBABLE | Medium | Separate device required |

### Recommended: Strategy A (Short-term) + D (Long-term)

**Strategy A: Linux VM Bridge**
```
┌─────────────────────────────────────────┐
│            Windows Host                  │
│                                          │
│  ┌──────────────────┐  ┌──────────────┐ │
│  │ UniversalDrop    │  │  Linux VM    │ │
│  │ Windows App      │  │  (AWDL +     │ │
│  │ (native protocol)│  │   AirDrop    │ │
│  │                  │  │   receiver)  │ │
│  └────────┬─────────┘  └──────┬───────┘ │
│           │                    │         │
│           │◄── Native ────────│         │
│           │    Protocol       │         │
│           │                   │         │
│  ┌────────┴──────────────────┴────────┐ │
│  │        Shared Network               │ │
│  └─────────────────────────────────────┘ │
│                                          │
│  USB Wi-Fi Adapter ──► passed to VM      │
└──────────────────────────────────────────┘

Flow:
  iPhone → AirDrop → VM (Linux receiver) → Native protocol → Windows app
```

---

## 4. Windows App Architecture

### Rust Core + UI Options

| Option | Framework | Pros | Cons |
|--------|-----------|------|------|
| **egui** | Pure Rust | Single language; fast dev | Not "Windows-native" look |
| **WinUI 3** | C# + XAML | Native look; system integration | Requires C#/Rust bridge |
| **Tauri** | Web (HTML/JS) | Familiar UI tech | Web overhead; less native |
| **iced** | Pure Rust | Elm-inspired; clean | Less mature |

**Recommendation:** Start with **egui** for MVP; migrate to **WinUI 3** for production polish.

### System Integration

| Feature | API | Notes |
|---------|-----|-------|
| System tray icon | `Shell_NotifyIcon` / WinRT | Always-available presence |
| Toast notifications | `Windows.UI.Notifications` | Incoming transfer requests |
| File explorer context menu | Shell extension | "Send via UniversalDrop" |
| Startup on login | Registry `Run` key | Optional auto-start |
| Windows Service | `sc.exe` / `windows-service` crate | Background receiving |
| Credential storage | `Windows Credential Manager` | Device keys via DPAPI |
| Firewall | Windows Firewall API | Auto-configure inbound rule |

### Background Operation

```
Windows Service ("UniversalDrop Receiver"):
  - Runs as Local Service account
  - Starts on system boot
  - Listens for incoming connections
  - Shows notification via toast
  - Communicates with UI app via named pipe / localhost TCP

UI Application:
  - Tray icon when minimized
  - Shows nearby devices
  - Handles send/receive UI
  - Communicates with service for background operations
```

---

## 5. Rust on Windows

### windows-rs Crate

```rust
use windows::Devices::Bluetooth::Advertisement::*;

// BLE Advertisement
let publisher = BluetoothLEAdvertisementPublisher::new()?;
let data = BluetoothLEManufacturerData::new()?;
data.SetCompanyId(0x004C)?;  // Apple
// Set payload bytes...
publisher.Advertisement()?.ManufacturerData()?.Append(&data)?;
publisher.Start()?;
```

### Build Requirements
- Windows SDK (10.0.19041.0 or later)
- Visual Studio Build Tools (MSVC)
- Rust target: `x86_64-pc-windows-msvc`

---

## 6. Windows Version Support

| Windows Version | Build | BLE | mDNS | Wi-Fi Direct | Status |
|----------------|-------|-----|------|-------------|--------|
| Windows 10 1703+ | 15063+ | ✅ | ✅ | ✅ | ✅ Supported |
| Windows 10 22H2 | 19045 | ✅ | ✅ | ✅ | ✅ Recommended minimum |
| Windows 11 23H2 | 22631 | ✅ | ✅ | ✅ | ✅ Recommended |
| Windows 11 24H2 | 26100 | ✅ | ✅ | ✅ | ✅ Latest |
| Windows 10 pre-1703 | <15063 | ❌ | ❌ | ⚠️ | 🔴 Not supported |

---

## 7. Recommendations

1. **Phase 6:** Build Windows app with native UniversalDrop protocol
2. **System tray + Windows Service** for background receiving
3. **egui** for initial UI; evaluate WinUI 3 for production
4. **AirDrop:** Linux VM bridge as documented workaround (Phase 8)
5. **Monitor DMA:** Wi-Fi Aware may provide native AirDrop path in future
6. **Target Windows 10 22H2+** as minimum supported version
