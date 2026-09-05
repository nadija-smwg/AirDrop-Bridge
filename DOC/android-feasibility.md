# UniversalDrop — Android Feasibility Report

**Date:** September 2026

---

## 1. Summary

| Aspect | Status |
|--------|--------|
| Native UniversalDrop protocol | ✅ Fully feasible |
| AirDrop compatibility (third-party) | 🔴 NOT FEASIBLE without chipset vendor support |
| AirDrop via Wi-Fi Aware (DMA, future) | 🟡 PROBABLE when Apple ships support |
| AirDrop via relay bridge | 🟡 PROBABLE with Linux intermediary |

---

## 2. Available APIs

### Bluetooth LE

| API | Min SDK | Permission | Purpose |
|-----|---------|------------|---------|
| `BluetoothLeScanner` | 21 (5.0) | `BLUETOOTH_SCAN` (31+) | Scan for nearby BLE devices |
| `BluetoothLeAdvertiser` | 21 (5.0) | `BLUETOOTH_ADVERTISE` (31+) | Broadcast BLE advertisements |
| `BluetoothGatt` | 18 (4.3) | `BLUETOOTH_CONNECT` (31+) | GATT connections |

**Feasibility:** ✅ Full BLE control available
**Caveat:** Some vendors may filter custom manufacturer data (Company ID 0x004C)

### Wi-Fi

| API | Min SDK | Permission | Purpose |
|-----|---------|------------|---------|
| `WifiManager` | 1 | `ACCESS_WIFI_STATE` | Wi-Fi state management |
| `WifiP2pManager` | 14 (4.0) | `NEARBY_WIFI_DEVICES` (33+) | Wi-Fi Direct P2P |
| `WifiAwareManager` | 26 (8.0) | `NEARBY_WIFI_DEVICES` (33+) | Wi-Fi Aware (NAN) |
| `NsdManager` | 16 (4.1) | None | mDNS service discovery |

**Wi-Fi Aware Status:** ✅ Available on modern flagships
- Pixel 6+, Samsung Galaxy S21+, most 2022+ flagships
- Hardware-dependent (not all devices support it)
- Supports direct device-to-device communication without AP

### Network

| API | Purpose | Notes |
|-----|---------|-------|
| `Socket` / `ServerSocket` | TCP connections | Standard Java/Kotlin sockets |
| `SSLSocket` | TLS connections | Via `javax.net.ssl` |
| `NsdManager` | mDNS | Service registration and discovery |

### File System

| API | Min SDK | Purpose |
|-----|---------|---------|
| `MediaStore` | 29 (10) | Access shared media (scoped storage) |
| `SAF (DocumentFile)` | 21 (5.0) | User-selected directories |
| `Environment.getExternalStoragePublicDirectory` | Deprecated | Legacy file access |

**Scoped Storage:** ⚠️ Android 10+ enforces scoped storage
- Must use `MediaStore` or SAF for saving received files
- Cannot write to arbitrary filesystem locations
- Downloads folder accessible via `MediaStore.Downloads`

### Background Execution

| Component | Purpose | Requirement |
|-----------|---------|-------------|
| Foreground Service | Keep receiving in background | Persistent notification required |
| `TYPE_CONNECTED_DEVICE` | Service type for device connectivity | Android 14+ |
| WorkManager | Periodic tasks | Not suitable for real-time receiving |

---

## 3. Rust Integration

### NDK / JNI Bridge

```
Android App Architecture:

┌──────────────────────────┐
│   Kotlin UI Layer         │
│   (Jetpack Compose)       │
├──────────────────────────┤
│   Kotlin ↔ Rust Bridge    │
│   (JNI via `jni` crate)   │
├──────────────────────────┤
│   Rust Core Library       │
│   (.so shared library)    │
│   ud-protocol             │
│   ud-transfer             │
│   ud-crypto               │
└──────────────────────────┘
```

**Cross-compilation targets:**
```
aarch64-linux-android    # ARM64 (most modern phones)
armv7-linux-androideabi  # ARM32 (older phones)
x86_64-linux-android     # Emulator
```

**Build tool:** `cargo-ndk` for cross-compilation

---

## 4. AWDL on Android

### Current State: 🔴 NOT FEASIBLE for third-party apps

**Why:**
- Android does not expose monitor mode to userspace (even with root on most devices)
- Wi-Fi chipset firmware is vendor-locked (Qualcomm, MediaTek, Samsung)
- No raw 802.11 frame injection API exists in Android
- Google's Pixel 10 AirDrop (Quick Share) uses custom firmware-level AWDL:
  - Modified Qualcomm Wi-Fi firmware
  - Proprietary HAL extensions
  - Not available to third-party apps
  - Not available to non-Pixel devices (initially)

### Possible Paths to Android AirDrop

| Path | Feasibility | Timeline |
|------|-------------|----------|
| **Wi-Fi Aware (DMA)** | 🟡 PROBABLE | When Apple implements NAN for AirDrop (2026-2027?) |
| **Root + custom driver** | 🧪 EXPERIMENTAL | Now, but negligible audience |
| **Linux relay bridge** | 🟡 PROBABLE | Now, requires Linux intermediary |
| **OEM partnership** | ❓ UNKNOWN | Requires vendor cooperation |

---

## 5. Permissions Model (Android 14+)

### Required Permissions
```xml
<!-- Bluetooth -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />

<!-- Wi-Fi -->
<uses-permission android:name="android.permission.NEARBY_WIFI_DEVICES"
    android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />

<!-- Network -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

<!-- Storage -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />

<!-- Foreground Service -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

### Runtime Permission Flow
1. Request Bluetooth permissions on first launch
2. Request nearby Wi-Fi permissions
3. Request notification permission
4. Request storage permission when saving files

---

## 6. Recommendations

1. **Phase 5:** Build Android app with native UniversalDrop protocol only
2. **Phase 7:** Add AirDrop support when Wi-Fi Aware path becomes available
3. **Interim:** Provide relay bridge documentation (Linux intermediary)
4. **Target Android 12+ (API 31+)** for modern permission model
5. **Test on multiple vendors** to catch BLE/Wi-Fi API variations
