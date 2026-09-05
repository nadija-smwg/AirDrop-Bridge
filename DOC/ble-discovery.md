# UniversalDrop — BLE Discovery Research

**Date:** September 2026

---

## 1. How AirDrop BLE Discovery Works

### Overview
When a user opens the AirDrop sharing sheet on an iPhone, the device begins scanning for BLE advertisements from potential receivers. Receivers that want to appear in the AirDrop device list must broadcast specific BLE advertisements.

### Flow
```
1. User opens Share → AirDrop on iPhone
2. iPhone begins active BLE scanning
3. iPhone also broadcasts its own BLE advertisement with sender identity hash
4. Nearby receivers detect the scan/broadcast
5. Receivers respond by advertising their availability
6. iPhone discovers receiver and shows it in the device list
```

---

## 2. BLE Advertisement Format

### Apple Manufacturer Data Structure
```
┌─────────────────────────────────────────────────┐
│ AD Structure (BLE Advertisement Data)            │
├──────────┬──────────────────────────────────────┤
│ Length   │ Total length of this AD structure      │
│ Type     │ 0xFF (Manufacturer Specific Data)      │
│ Company  │ 0x4C 0x00 (Apple Inc., little-endian)  │
├──────────┼──────────────────────────────────────┤
│ Sub-type │ 0x05 (AirDrop)                         │
│ Length   │ Length of AirDrop-specific data         │
├──────────┼──────────────────────────────────────┤
│ Flags    │ Status/capability flags                │
│ Hash     │ Short identity hash (2+ bytes)          │
│ Apple ID │ Truncated Apple ID hash                │
│ Phone    │ Truncated phone number hash            │
│ Email    │ Truncated email hash                   │
│ AWDL     │ AWDL state information                 │
└──────────┴──────────────────────────────────────┘
```

### Key Fields

| Field | Size | Purpose |
|-------|------|---------|
| Company ID | 2 bytes | Identifies Apple (0x4C00) |
| Sub-type | 1 byte | 0x05 = AirDrop |
| Status | 1 byte | Availability flags |
| Short Hash | 2 bytes | First 2 bytes of SHA-256(contact_info) |
| Apple ID Hash | 2 bytes | Truncated hash of Apple ID |
| Phone Hash | 2 bytes | Truncated hash of phone number |
| Email Hash | 2 bytes | Truncated hash of email |

---

## 3. "Contacts Only" vs "Everyone" BLE Behavior

### "Contacts Only" Mode
1. Receiver broadcasts BLE with its identity hashes
2. Sender compares hashes against its contact list
3. If match found → proceed to AWDL connection
4. If no match → receiver is invisible to sender

### "Everyone" Mode  
1. Receiver broadcasts BLE with generic/empty identity hash
2. Sender always proceeds to AWDL connection regardless of hash match
3. No contact list comparison needed

### Implication for UniversalDrop
- In "Everyone" mode, we can use **any** or **empty** identity hashes
- The receiver just needs to be visible; contact matching is irrelevant
- This is why all open implementations only work with "Everyone" mode

---

## 4. Platform Implementation

### Linux
```
Tool: BlueZ / hcitool / btmgmt
API:  HCI (Host Controller Interface)

# Scan for Apple BLE advertisements
sudo hcitool lescan

# Set custom advertising data (raw HCI)
sudo hcitool -i hci0 cmd 0x08 0x0008 \
  1E 02 01 06 1A FF 4C 00 05 ...

# Using btmgmt
sudo btmgmt add-adv -d "1AFF4C000512..." 1
```

**Status:** ✅ CONFIRMED — Full control over BLE advertising data

### Android
```
API: BluetoothLeAdvertiser

val advertiser = bluetoothAdapter.bluetoothLeAdvertiser
val settings = AdvertiseSettings.Builder()
    .setAdvertiseMode(AdvertiseSettings.ADVERTISE_MODE_LOW_LATENCY)
    .setConnectable(false)
    .build()
    
val data = AdvertiseData.Builder()
    .addManufacturerData(0x004C, byteArrayOf(0x05, ...))
    .build()
    
advertiser.startAdvertising(settings, data, callback)
```

**Status:** 🧪 EXPERIMENTAL
- `addManufacturerData(0x004C, ...)` may be rejected by some Android vendors
- Some manufacturers filter Apple Company ID
- Needs testing on multiple devices

### Windows
```
API: Windows.Devices.Bluetooth.Advertisement.BluetoothLEAdvertisementPublisher

var publisher = new BluetoothLEAdvertisementPublisher();
var data = new BluetoothLEManufacturerData();
data.CompanyId = 0x004C;  // Apple
data.Data = /* AirDrop payload bytes */;
publisher.Advertisement.ManufacturerData.Add(data);
publisher.Start();
```

**Status:** 🧪 EXPERIMENTAL
- WinRT API supports custom manufacturer data
- Need to verify Windows allows using Apple's Company ID
- Bluetooth driver support varies

---

## 5. Privacy Concerns

### Hash Reversal Attack (hexway research)
- AirDrop uses truncated SHA-256 hashes of contact information
- With only 2 bytes, there are only 65,536 possible values
- An attacker can pre-compute hashes for all phone numbers in a region
- Matching a 2-byte hash reveals the contact's phone number

### UniversalDrop Mitigation
- Native protocol does NOT use contact hashes
- AirDrop compatibility mode uses empty/random hashes
- BLE advertisement contains minimal identifying information
- MAC address rotation where supported

---

## 6. Open Questions

| Question | Status |
|----------|--------|
| Does iOS validate that the BLE Company ID is genuinely from an Apple device? | ❓ UNKNOWN — needs testing |
| Does iOS check a Bluetooth SIG database or Apple-specific certificate? | ❓ UNKNOWN |
| What is the exact payload format for iOS 18 AirDrop BLE? | 🧪 Needs packet capture |
| Do any Android vendors block manufacturer data with Company ID 0x004C? | 🧪 Needs testing |
| Does Windows allow BLE advertising with Company ID 0x004C? | 🧪 Needs testing |
| Is the BLE advertisement required, or can AWDL work without it? | 🟡 AWDL can work without BLE but iPhone may not discover receiver |
