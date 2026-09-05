# UniversalDrop — Native Protocol Design

**Date:** September 2026

---

## 1. Protocol Overview

The UniversalDrop native protocol provides secure, resumable file transfer between Android, Windows, and Linux devices over local Wi-Fi.

**Transport:** TCP + TLS 1.3
**Discovery:** mDNS (`_universaldrop._tcp.local.`)
**Encoding:** Custom binary format with HMAC integrity
**Identity:** Ed25519 device keys, TOFU certificate pinning

---

## 2. Message Header

Every message begins with a fixed 12-byte header:

```
Offset  Size    Field           Description
0       4       magic           "UDRP" (0x55445250)
4       1       version         Protocol version (1)
5       1       msg_type        Message type ID
6       2       flags           Bit flags
8       4       payload_len     Payload length in bytes (big-endian)
```

Followed by:
```
12      N       payload         Message payload (N = payload_len)
12+N    32      hmac            HMAC-SHA256 of header + payload
```

Total message size: 12 + payload_len + 32 bytes

### Flags
```
Bit 0: COMPRESSED   — payload is zstd compressed
Bit 1: ENCRYPTED    — payload is additionally encrypted (beyond TLS)
Bit 2: CHUNKED      — part of a multi-chunk sequence
Bit 3-15: Reserved
```

---

## 3. Message Types

### Discovery & Connection

| ID | Name | Direction | Purpose |
|----|------|-----------|---------|
| `0x01` | HELLO | Both | Version negotiation, capabilities |
| `0x02` | DEVICE_INFO | Both | Exchange device metadata |

### Pairing

| ID | Name | Direction | Purpose |
|----|------|-----------|---------|
| `0x03` | PAIR_REQUEST | Initiator | Request device pairing |
| `0x04` | PAIR_RESPONSE | Responder | Accept/reject pairing |
| `0x05` | AUTH | Both | Authentication challenge/response |

### Transfer

| ID | Name | Direction | Purpose |
|----|------|-----------|---------|
| `0x10` | TRANSFER_REQUEST | Sender | Propose file transfer |
| `0x11` | TRANSFER_ACCEPT | Receiver | Accept transfer |
| `0x12` | TRANSFER_REJECT | Receiver | Reject transfer |

### File Data

| ID | Name | Direction | Purpose |
|----|------|-----------|---------|
| `0x20` | FILE_METADATA | Sender | File info (name, size, hash, type) |
| `0x21` | FILE_CHUNK | Sender | File data chunk |
| `0x22` | FILE_ACK | Receiver | Acknowledge chunk(s) |
| `0x23` | FILE_RESUME | Both | Resume from offset |
| `0x24` | FILE_COMPLETE | Sender | All chunks sent |
| `0x25` | HASH_VERIFY | Receiver | Full file hash result |

### Control

| ID | Name | Direction | Purpose |
|----|------|-----------|---------|
| `0x30` | TRANSFER_COMPLETE | Receiver | Transfer confirmed successful |
| `0xF0` | CANCEL | Both | Cancel current operation |
| `0xFF` | ERROR | Both | Error with code and message |

---

## 4. Message Payloads

### HELLO (0x01)
```json
{
    "protocol_version": 1,
    "min_version": 1,
    "device_id": "<Ed25519 public key fingerprint>",
    "capabilities": ["transfer", "resume", "compress", "multi_file"],
    "nonce": "<32 random bytes>"
}
```

### DEVICE_INFO (0x02)
```json
{
    "device_name": "My Android Phone",
    "device_type": "phone",        // phone | tablet | desktop | laptop
    "platform": "android",         // android | windows | linux
    "platform_version": "14",
    "app_version": "1.0.0",
    "cert_fingerprint": "<SHA-256 of TLS certificate>"
}
```

### TRANSFER_REQUEST (0x10)
```json
{
    "transfer_id": "<UUID>",
    "files": [
        {
            "file_id": "<UUID>",
            "name": "photo.jpg",
            "size": 8421376,
            "mime_type": "image/jpeg",
            "sha256": "<hex hash>",
            "modified": "2026-09-05T14:30:00Z"
        }
    ],
    "total_size": 8421376,
    "total_files": 1
}
```

### FILE_METADATA (0x20)
```json
{
    "transfer_id": "<UUID>",
    "file_id": "<UUID>",
    "name": "photo.jpg",
    "size": 8421376,
    "mime_type": "image/jpeg",
    "sha256": "<hex hash>",
    "chunk_size": 1048576,
    "total_chunks": 9
}
```

### FILE_CHUNK (0x21)
```
Binary payload:
  Offset  Size    Field
  0       16      transfer_id (UUID bytes)
  16      16      file_id (UUID bytes)
  32      4       chunk_index (big-endian u32)
  36      4       chunk_length (big-endian u32)
  40      N       chunk_data (raw file bytes)
```

### FILE_ACK (0x22)
```json
{
    "transfer_id": "<UUID>",
    "file_id": "<UUID>",
    "acked_chunk": 5,
    "received_bytes": 6291456
}
```

### FILE_RESUME (0x23)
```json
{
    "transfer_id": "<UUID>",
    "file_id": "<UUID>",
    "resume_from_chunk": 6400,
    "resume_from_byte": 6710886400,
    "partial_sha256": "<hash of bytes received so far>"
}
```

### ERROR (0xFF)
```json
{
    "code": 4001,
    "message": "Transfer rejected by user",
    "transfer_id": "<UUID>"
}
```

### Error Codes
| Code | Meaning |
|------|---------|
| 1000 | Protocol version mismatch |
| 1001 | Invalid message format |
| 2000 | Authentication failed |
| 2001 | Pairing rejected |
| 3000 | Transfer rejected |
| 3001 | File too large |
| 3002 | Insufficient storage |
| 3003 | Hash verification failed |
| 3004 | Resume failed |
| 4000 | Internal error |
| 4001 | User cancelled |
| 4002 | Timeout |

---

## 5. Transfer State Machine

```
       ┌──────────┐
       │  IDLE     │
       └────┬─────┘
            │ HELLO exchange
       ┌────▼─────┐
       │ CONNECTED │
       └────┬─────┘
            │ TRANSFER_REQUEST
       ┌────▼──────────┐
       │ AWAITING_ACCEPT│
       └────┬──────────┘
            │ TRANSFER_ACCEPT
       ┌────▼─────┐         TRANSFER_REJECT
       │TRANSFERRING├──────────────►┌──────┐
       └────┬─────┘                 │ IDLE │
            │                       └──────┘
            │ for each file:
            │   FILE_METADATA
            │   FILE_CHUNK (x N)
            │   FILE_ACK (x N)
            │   FILE_COMPLETE
            │   HASH_VERIFY
            │
       ┌────▼────────────┐
       │TRANSFER_COMPLETE │
       └────┬────────────┘
            │
       ┌────▼─────┐
       │  IDLE     │
       └──────────┘
```

---

## 6. Resume Protocol

```
Connection Lost During Transfer:

1. Sender detects connection failure
2. Sender saves transfer state:
   - transfer_id, file_id
   - last acknowledged chunk
   - partial hash of sent data

3. Receiver saves transfer state:
   - transfer_id, file_id
   - last received chunk
   - partial hash of received data
   - temp file with partial data

4. Reconnection:
   a. Re-establish TCP + TLS connection
   b. HELLO exchange
   c. Sender sends FILE_RESUME {chunk, byte_offset, partial_hash}
   d. Receiver verifies partial_hash matches its data
   e. If match: FILE_RESUME response with confirmed offset
   f. If mismatch: restart from beginning
   g. Transfer continues from confirmed offset
```

---

## 7. Discovery

### mDNS Service
```
Service Name:  _universaldrop._tcp.local.
Port:          Dynamic (e.g., 42000-42999)
Host:          <device-name>.local.
TXT Records:
  v=1                          # Protocol version
  id=<fingerprint first 8>     # Short device ID
  name=<device name>           # Display name
  type=phone                   # Device type
  caps=transfer,resume         # Capabilities
```

### Discovery Flow
```
Device A                        Device B
   │                                │
   │── mDNS Query ────────────────► │
   │   _universaldrop._tcp          │
   │                  ◄── mDNS Resp─│
   │   service records              │
   │                                │
   │── TCP Connect ───────────────► │
   │── TLS Handshake ─────────────► │
   │── HELLO ─────────────────────► │
   │                  ◄── HELLO ────│
   │── DEVICE_INFO ───────────────► │
   │             ◄── DEVICE_INFO ──│
   │                                │
   │ [Device B shown in device list]│
```

---

## 8. Chunking Strategy

```
Default chunk size:  1 MB (1,048,576 bytes)
Minimum chunk size:  64 KB
Maximum chunk size:  16 MB
Configurable:        Yes (negotiated in HELLO capabilities)

File: 10 GB (10,737,418,240 bytes)
Chunks: 10,240 (at 1 MB each)
Last chunk: 10,240th chunk may be smaller

ACK strategy: Every chunk (can batch-ACK for performance)
Window: Up to 16 unacknowledged chunks (configurable)
```

---

## 9. Compression

```
Algorithm:    zstd (level 3, default)
Applied:      Per-chunk, optional
Signaled:     COMPRESSED flag in message header
Threshold:    Only compress if chunk > 64 KB
Skip:         Already-compressed formats (JPEG, MP4, ZIP, etc.)
Detection:    MIME type or file extension heuristic
```
