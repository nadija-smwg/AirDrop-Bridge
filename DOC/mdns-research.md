# UniversalDrop — mDNS Research

**Date:** September 2026

---

## 1. Role in AirDrop

mDNS (Multicast DNS) / Bonjour is used for service discovery after the AWDL peer-to-peer link is established. It allows the sender to find the receiver's AirDrop HTTPS server.

**Service:** `_airdrop._tcp.local.`

---

## 2. How It Works

```
1. Receiver registers mDNS service on AWDL interface
2. Sender queries for _airdrop._tcp.local.
3. Receiver responds with:
   - SRV record (hostname + port)
   - TXT record (capabilities)
   - AAAA record (IPv6 link-local address)
4. Sender resolves hostname → IPv6 address
5. Sender connects to HTTPS server on that address:port
```

### Service Record Format
```
Name:    <instance-name>._airdrop._tcp.local.
Type:    SRV
Port:    <dynamic, e.g., 8770>
Host:    <hostname>.local.
TXT:     flags=<hex capability flags>
Address: fe80::xxxx:xxxx:xxxx:xxxx (IPv6 link-local)
```

---

## 3. Native UniversalDrop mDNS

For the native protocol, we use a separate service type:

```
Name:    <device-name>._universaldrop._tcp.local.
Type:    SRV
Port:    <dynamic>
Host:    <hostname>.local.
TXT:
  v=1              # Protocol version
  id=<fingerprint> # Short device ID (first 8 chars)
  name=<name>      # Display name
  type=phone       # Device type (phone/tablet/desktop/laptop)
  caps=transfer,resume,compress
```

---

## 4. Platform Support

| Platform | Implementation | Status |
|----------|---------------|--------|
| **Linux** | Avahi (system service) or manual multicast | ✅ Full support |
| **Android** | `NsdManager` API | ✅ Full support |
| **Windows** | Native DNS client (Win10 1703+) | ✅ Resolution supported; registration may need library |

### Rust Libraries

| Crate | License | Features |
|-------|---------|----------|
| `mdns-sd` | Apache-2.0/MIT | Service discovery and registration |
| `zeroconf` | MIT | Cross-platform Bonjour/Avahi wrapper |
| `trust-dns-resolver` | Apache-2.0/MIT | DNS resolution (including mDNS) |

---

## 5. Key Consideration

> mDNS for AirDrop must operate on the **AWDL virtual interface**, not on the standard Wi-Fi interface. This means the AWDL link layer must be working first. mDNS is the easy part; AWDL is the hard part.

For the native UniversalDrop protocol, mDNS operates on the **standard Wi-Fi interface** — no AWDL needed.
