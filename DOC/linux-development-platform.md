# UniversalDrop — Linux Development Platform

**Date:** September 2026

---

## 1. Why Linux First

Linux is the only platform where AWDL can be implemented in userspace with commodity hardware:
- Monitor mode Wi-Fi is well-supported
- Packet injection works with compatible chipsets
- BlueZ provides full BLE control
- All tools (Wireshark, tcpdump, hcitool) are available
- Both OWL and opendrop-rs are Linux-first

---

## 2. Recommended Distribution

| Option | Pros | Cons | Recommended |
|--------|------|------|-------------|
| **Ubuntu 24.04 LTS** | Stable, long-term support, large community | May need manual Wi-Fi driver install | ✅ Primary |
| **Kali Linux** | Pre-installed wireless tools, OWL in repos | Rolling release, less stable for dev | 🟡 Alternative |
| **Fedora** | Modern kernel, good hardware support | Shorter support cycle | 🟡 Alternative |

---

## 3. AirDrop Receiver CLI Tool

### Design
```
linux-receiver — AirDrop-compatible file receiver for Linux

USAGE:
    linux-receiver [OPTIONS]

OPTIONS:
    -i, --interface <IFACE>     Wi-Fi interface for AWDL (e.g., wlan1)
    -b, --bt-interface <IFACE>  Bluetooth interface (default: hci0)
    -n, --name <NAME>           Device name shown to senders (default: hostname)
    -d, --save-dir <DIR>        Directory to save received files (default: ~/Downloads/UniversalDrop)
    -l, --log-level <LEVEL>     Log level: error, warn, info, debug, trace
        --no-ble                Disable BLE advertising
        --auto-accept           Auto-accept all transfers (testing only)
    -h, --help                  Show help
```

### Structured Log Output
```
[2026-09-05 14:30:00] [INFO ] [INIT    ] UniversalDrop Linux Receiver v0.1.0
[2026-09-05 14:30:00] [INFO ] [INIT    ] Wi-Fi interface: wlan1 (Alfa AWUS036ACH, RTL8812AU)
[2026-09-05 14:30:00] [INFO ] [INIT    ] Bluetooth: hci0 (CSR8510)
[2026-09-05 14:30:00] [INFO ] [INIT    ] Save directory: /home/user/Downloads/UniversalDrop/
[2026-09-05 14:30:01] [INFO ] [BLE     ] Advertising started (Apple 0x004C, AirDrop 0x05)
[2026-09-05 14:30:01] [INFO ] [AWDL    ] Entering monitor mode on wlan1
[2026-09-05 14:30:01] [INFO ] [AWDL    ] Scanning social channels: 6, 44, 149
[2026-09-05 14:30:03] [INFO ] [AWDL    ] PSF received from master (iPhone) on ch 44
[2026-09-05 14:30:03] [INFO ] [AWDL    ] Synced to master. AW: 16 TU, offset: 0
[2026-09-05 14:30:03] [INFO ] [MDNS    ] Registered _airdrop._tcp on port 8770
[2026-09-05 14:30:05] [INFO ] [TLS     ] Connection from fe80::xxxx:xxxx:xxxx:xxxx
[2026-09-05 14:30:05] [INFO ] [AIRDROP ] POST /Discover from "iPhone 15 Pro"
[2026-09-05 14:30:05] [INFO ] [AIRDROP ] Responded: ReceiverComputerName="UniversalDrop"
[2026-09-05 14:30:08] [INFO ] [AIRDROP ] POST /Ask: IMG_4821.HEIC (8.4 MB)
[2026-09-05 14:30:08] [INFO ] [TRANSFER] 
[2026-09-05 14:30:08] [INFO ] [TRANSFER] ┌────────────────────────────────────┐
[2026-09-05 14:30:08] [INFO ] [TRANSFER] │ Incoming AirDrop Request           │
[2026-09-05 14:30:08] [INFO ] [TRANSFER] │ From: iPhone 15 Pro                │
[2026-09-05 14:30:08] [INFO ] [TRANSFER] │ File: IMG_4821.HEIC (8.4 MB)       │
[2026-09-05 14:30:08] [INFO ] [TRANSFER] │                                    │
[2026-09-05 14:30:08] [INFO ] [TRANSFER] │ Accept? [Y/n]                      │
[2026-09-05 14:30:08] [INFO ] [TRANSFER] └────────────────────────────────────┘
[2026-09-05 14:30:10] [INFO ] [TRANSFER] Accepted by user
[2026-09-05 14:30:10] [INFO ] [AIRDROP ] POST /Upload started
[2026-09-05 14:30:11] [INFO ] [TRANSFER] Receiving: ████████░░░░░░░░ 45% (3.8 / 8.4 MB)
[2026-09-05 14:30:13] [INFO ] [TRANSFER] Receiving: ████████████████ 100% (8.4 / 8.4 MB)
[2026-09-05 14:30:13] [INFO ] [SECURITY] SHA-256 verified: OK
[2026-09-05 14:30:13] [INFO ] [TRANSFER] Saved: /home/user/Downloads/UniversalDrop/IMG_4821.HEIC
[2026-09-05 14:30:13] [INFO ] [TRANSFER] Transfer complete ✓
```

---

## 4. Required Packages

```bash
# Ubuntu 24.04
sudo apt update
sudo apt install -y \
  build-essential \
  pkg-config \
  libssl-dev \
  libdbus-1-dev \
  libbluetooth-dev \
  libpcap-dev \
  wireless-tools \
  aircrack-ng \
  wireshark \
  tshark \
  bluez \
  bluetooth \
  git \
  curl

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

---

## 5. Packet Capture Lab Setup

```bash
# 1. Enable monitor mode
sudo airmon-ng check kill
sudo airmon-ng start wlan1

# 2. Capture on social channel 6
sudo tcpdump -i wlan1mon -w awdl_ch6.pcap -c 1000

# 3. Capture on social channel 44 (5 GHz)
sudo iw dev wlan1mon set channel 44
sudo tcpdump -i wlan1mon -w awdl_ch44.pcap -c 1000

# 4. BLE capture
sudo hcitool lescan --duplicates &
sudo hcidump --raw > ble_capture.txt

# 5. Open Wireshark with AWDL dissector
wireshark -i wlan1mon -k
```
