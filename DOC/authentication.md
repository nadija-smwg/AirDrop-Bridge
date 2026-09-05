# UniversalDrop — Authentication & Identity Research

**Date:** September 2026

---

## 1. Three Layers of Identity

```
┌─────────────────────────────────┐
│     Layer 3: User Authorization  │
│     "Do I accept this file?"     │
│     → User taps Accept/Decline   │
├─────────────────────────────────┤
│     Layer 2: Device Auth         │
│     "Is this device legitimate?" │
│     → TLS certs, identity records│
├─────────────────────────────────┤
│     Layer 1: Device Discovery    │
│     "Can I see this device?"     │
│     → BLE, mDNS, AWDL           │
└─────────────────────────────────┘
```

These are independent concerns. A device can be discoverable without being authenticated, and authentication does not imply authorization.

---

## 2. AirDrop Identity Mechanism

### Apple Identity Record
- When user signs into iCloud, device generates RSA 2048-bit identity keypair
- Apple's servers sign a validation record for the device
- This `SenderRecordData` / `ReceiverRecordData` is exchanged during /Discover

### What Apple Signs
```
Apple-Signed Identity Record contains:
  - Device's public key
  - Associated Apple ID
  - Associated email addresses
  - Associated phone numbers
  - Apple's CA signature

This record proves:
  "Apple certifies that this public key belongs to this Apple ID"
```

### "Contacts Only" Requires Apple Signing
```
Contacts Only Flow:
1. Sender broadcasts short hash of contact info via BLE
2. Receiver checks if hash matches any contact
3. If match → respond, proceed to AWDL
4. During /Discover, exchange Apple-signed identity records
5. Verify signature against Apple's CA
6. Confirm that the sender's Apple ID matches a contact

Without Apple-signed record → "Contacts Only" CANNOT work
```

### "Everyone" Does NOT Require Apple Signing
```
Everyone Flow:
1. Sender broadcasts BLE (hash can be empty)
2. Receiver always responds
3. During /Discover, SenderRecordData can be empty or self-signed
4. No Apple CA verification needed
5. Receiver shows sender's name (unverified)
6. User decides whether to accept

With self-signed cert → "Everyone" WORKS ✅
```

---

## 3. What UniversalDrop Can and Cannot Do

| Feature | Without Apple Certs | With Apple Certs |
|---------|:-------------------:|:----------------:|
| Appear in AirDrop list ("Everyone") | ✅ | ✅ |
| Receive files ("Everyone") | ✅ | ✅ |
| Appear in AirDrop list ("Contacts Only") | ❌ | ✅ |
| Receive files ("Contacts Only") | ❌ | ✅ |
| Receive shared web links | ❌ | ✅ |
| Send files to iPhone | 🧪 Partial | ✅ |
| Show verified sender name | ❌ | ✅ |

---

## 4. UniversalDrop Native Identity

### Device Identity (Ed25519)
```
Key Generation:
  - Ed25519 keypair generated on first app launch
  - Private key stored in OS secure storage
  - Public key used for device fingerprint

Device Fingerprint:
  - SHA-256(public_key)[0..16] → 32 hex chars
  - Displayed as: XXXX-XXXX-XXXX-XXXX
  - Used for visual verification during pairing
```

### TLS Certificate
```
Self-Signed Certificate:
  - Subject: CN=<device-name>, O=UniversalDrop
  - Key: Ed25519 (or RSA 2048 for broader compatibility)
  - Validity: 10 years (long-lived; device-specific)
  - Serial: Random 128-bit
  - Extensions: SubjectAltName with device fingerprint
```

### TOFU (Trust On First Use)
```
First Connection:
  1. Devices exchange TLS certificates
  2. Each device displays the other's fingerprint
  3. User visually verifies (optional but recommended)
  4. Certificate is pinned (saved to local database)

Subsequent Connections:
  1. TLS handshake presents certificate
  2. Certificate compared against pinned cert
  3. If match → connection trusted
  4. If mismatch → WARNING: possible impersonation
```

### Pairing (Optional Enhancement)
```
Pairing Process:
  1. Device A displays 6-digit code
  2. User enters code on Device B
  3. Code confirms both devices see the same connection
  4. Certificates mutually pinned
  5. Future connections auto-trusted

Paired devices bypass the accept/decline dialog for trusted senders.
```

---

## 5. Security Analysis

| Threat | AirDrop (Everyone) | UniversalDrop (TOFU) |
|--------|:------------------:|:--------------------:|
| MITM on first connection | ⚠️ Possible | ⚠️ Possible (mitigated by fingerprint display) |
| MITM on subsequent | ❌ Blocked by TLS | ❌ Blocked by cert pinning |
| Impersonation | ⚠️ Name can be spoofed | ⚠️ Name can be spoofed; fingerprint cannot |
| Identity verification | ❌ No verification in Everyone | ✅ Fingerprint + optional pairing |
| Replay | ❌ Blocked by TLS | ❌ Blocked by TLS + nonce |
