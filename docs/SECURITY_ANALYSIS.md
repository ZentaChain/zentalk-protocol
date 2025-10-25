# ZenTalk Security & Privacy Analysis

## Executive Summary

ZenTalk implements **true end-to-end encryption** with **forward secrecy**. Relay servers store and forward encrypted message blobs but **cannot decrypt any message content**. This document explains exactly what relay operators can and cannot see.

---

## 1. Message Encryption Architecture

### Encryption Layers

Messages are protected by **three layers of encryption**:

```
┌─────────────────────────────────────────────────────┐
│  Layer 3: Double Ratchet (E2EE)                     │
│  - Signal Protocol encryption                        │
│  - Unique key per message                            │
│  - Forward secrecy                                   │
│  └─────────────────────────────────────────────────┘│
│    ┌─────────────────────────────────────────────┐  │
│    │  Layer 2: Message Payload                   │  │
│    │  - From/To addresses                        │  │
│    │  - Timestamp                                │  │
│    │  - Content                                  │  │
│    └─────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────┐
│  Layer 1: Onion Routing                              │
│  - RSA-4096 encryption per relay hop                 │
│  - Each relay can only decrypt its own layer         │
│  - Relay only sees: next hop address                 │
└─────────────────────────────────────────────────────┘
```

### Code References

**Double Ratchet Encryption** (`pkg/network/message_sender.go:83`):
```go
// Encrypt message using ratchet
ratchetHeader, ciphertext, err := session.RatchetEncrypt(plaintext, AESEncryptGCM)
```

**Onion Layer Building** (`pkg/network/message_sender.go:106`):
```go
// Build onion layers around the ratchet payload
onion, err := crypto.BuildOnionLayers(relayPath, to, ratchetPayload)
```

---

## 2. What Relay Servers Store

### Storage Location
- **Path**: `./data/relay-{port}-queue.db` (SQLite database)
- **TTL**: 30 days (configurable)
- **Cleanup**: Automatic hourly cleanup of expired messages

### Database Schema
```sql
CREATE TABLE queued_messages (
    id INTEGER PRIMARY KEY,
    recipient_addr TEXT NOT NULL,        -- Hex-encoded address (visible)
    message_id TEXT UNIQUE NOT NULL,     -- Unique message ID (visible)
    encrypted_payload BLOB NOT NULL,     -- ENCRYPTED MESSAGE BLOB
    timestamp INTEGER NOT NULL,          -- When queued (visible)
    expires_at INTEGER NOT NULL,         -- Expiration time (visible)
    attempts INTEGER NOT NULL            -- Delivery attempts (visible)
);
```

**Source**: `pkg/storage/relay_queue.go:66-75`

### What Relay Operators CAN See:
- ✅ Recipient's wallet address (20 bytes)
- ✅ Message ID (16 bytes, random)
- ✅ Message size (bytes)
- ✅ Timestamp when message was relayed
- ✅ Number of delivery attempts
- ✅ When message expires

### What Relay Operators CANNOT See:
- ❌ Message content (encrypted with Double Ratchet)
- ❌ Sender's identity (hidden in onion routing)
- ❌ Message type (text, image, video, etc.)
- ❌ Media encryption keys
- ❌ Previous messages (forward secrecy)

---

## 3. End-to-End Encryption Details

### X3DH Key Agreement
Initial key exchange using Extended Triple Diffie-Hellman:
- **Identity Key**: Long-term Ed25519 key
- **Signed Pre-Key**: Medium-term X25519 key
- **One-Time Pre-Keys**: Single-use X25519 keys
- **Ephemeral Key**: Fresh X25519 key per session

**Result**: Shared secret that even the relay cannot compute

### Double Ratchet Protocol
Based on Signal's specification: https://signal.org/docs/specifications/doubleratchet/

**Features**:
- **Forward Secrecy**: Old messages cannot be decrypted if keys are compromised
- **Future Secrecy**: Future messages safe even if current key is leaked
- **Out-of-Order**: Handles message reordering gracefully
- **Key Rotation**: New key for every message

**Code**: `pkg/protocol/ratchet.go`

### Encryption Algorithms
- **Symmetric**: AES-256-GCM (authenticated encryption)
- **Key Exchange**: X25519 (Curve25519 ECDH)
- **Key Derivation**: HKDF-SHA256
- **Onion Routing**: RSA-4096-OAEP

---

## 4. Onion Routing Privacy

### How It Works
```
Sender → [Relay 1] → [Relay 2] → [Relay 3] → Recipient
```

Each relay:
1. Decrypts its own onion layer (RSA-4096)
2. Sees **only** the next hop address
3. Forwards the remaining encrypted payload
4. **Cannot** see the final recipient or message content

**Code**: `pkg/network/relay_handlers.go:74`
```go
// Decrypt onion layer
layer, err := crypto.DecryptOnionLayer(payload, rs.PrivateKey)
// Relay only gets: NextHop address + encrypted Payload
```

### Privacy Guarantees
- **Sender Anonymity**: Relay doesn't know the original sender
- **Recipient Privacy**: Intermediate relays don't know final destination
- **Content Privacy**: No relay can decrypt the message content
- **Traffic Analysis Resistance**: Relays can't correlate messages

---

## 5. Message Deletion & Privacy

### Client-Side Deletion
When a user deletes a message:

**API Server** (`pkg/api/database_messages.go:165`):
```go
func (db *DB) DeleteMessage(userAddr, peerAddr, messageID string) error {
    query := `DELETE FROM messages WHERE user_address = ? AND peer_address = ? AND id = ?`
    _, err := db.Conn.Exec(query, userAddr, peerAddr, messageID)
    return err
}
```

**What Gets Deleted**:
- ✅ Message from user's local database
- ✅ Message history in API server memory

**What DOES NOT Get Deleted**:
- ❌ Message from recipient's device
- ❌ Encrypted blobs on relay servers (already unreadable)
- ❌ Messages from other participants' devices

### Relay Server Queued Messages
If a message is still in the relay queue (recipient offline):
- Relay stores **encrypted blob** for up to 30 days
- After TTL expires, message is **automatically deleted**
- Even if relay operator tries to decrypt: **impossible** (no ratchet keys)

**Automatic Cleanup** (`pkg/storage/relay_queue.go:208`):
```go
func (q *RelayMessageQueue) cleanupExpiredMessages() {
    ticker := time.NewTicker(1 * time.Hour)
    // Deletes expired messages every hour
}
```

---

## 6. Account Deletion

### User Data Deletion
**API**: `POST /api/delete-account`

**What Gets Deleted** (`pkg/api/database_messages.go:202`):
```go
func (db *DB) DeleteUserData(userAddr string) error {
    // 1. Delete all messages
    DELETE FROM messages WHERE user_address = ?

    // 2. Delete user record
    DELETE FROM users WHERE wallet_address = ?

    // 3. Clean up session
    // 4. Clean up DHT key bundle
}
```

**Relay Server** (`pkg/storage/relay_queue.go:156`):
```go
func (q *RelayMessageQueue) DeleteMessagesForRecipient(recipientAddr protocol.Address) error {
    query := `DELETE FROM queued_messages WHERE recipient_addr = ?`
    // Deletes all queued encrypted blobs for this user
}
```

### What Happens:
1. **API Server**: Deletes user account, messages, and session
2. **Relay Server**: Deletes queued encrypted message blobs
3. **DHT Network**: Key bundle expires (no republishing)
4. **Other Users**: Keep their copy of conversation (E2EE design)

---

## 7. Metadata Minimization

### What ZenTalk DOES Minimize:
| Metadata | Protection |
|----------|-----------|
| Sender identity | Hidden via onion routing from relays |
| Recipient identity | Only visible to final relay |
| Message content | Encrypted with Double Ratchet |
| Message type | Encrypted inside payload |
| Social graph | DHT lookup doesn't reveal relationships |

### What ZenTalk CANNOT Hide (Fundamental Limitations):
| Metadata | Reason |
|----------|--------|
| Recipient address | Needed for delivery |
| Message size | Network-level information |
| Timing | When messages are sent |
| IP address | Use Tor/VPN for IP privacy |

---

## 8. Security Verification

### Test Script
Run the automated security test:
```bash
cd /Users/harun/Desktop/zentalk-web2025/zentalk/src/zentalk-protocol
./test_messaging.sh
```

### Manual Verification

**1. Check Relay Database** (relay operator perspective):
```bash
cd data/
sqlite3 relay-9001-queue.db

-- View queued messages (operator can only see metadata)
SELECT
    recipient_addr,
    message_id,
    length(encrypted_payload) as size_bytes,
    datetime(timestamp, 'unixepoch') as queued_at,
    datetime(expires_at, 'unixepoch') as expires_at
FROM queued_messages;
```

**What you'll see**:
```
recipient_addr|message_id|size_bytes|queued_at|expires_at
efeaa8e5...|a1b2c3d4...|1247|2025-01-15 10:30:00|2025-02-14 10:30:00
```

**What you WON'T see**:
- Message content (encrypted blob)
- Sender address
- Message type
- Any decrypted data

**2. Inspect Encrypted Payload**:
```bash
# Try to view the encrypted blob
sqlite3 relay-9001-queue.db "SELECT hex(encrypted_payload) FROM queued_messages LIMIT 1;"
```

**Result**: Random-looking hex data (AES-256-GCM ciphertext)

**3. Network Traffic Analysis**:
```bash
# Monitor relay traffic (requires tcpdump/wireshark)
tcpdump -i lo0 -X port 9001
```

**What you'll see**:
- Encrypted onion-routed packets
- Protocol headers (magic bytes, version, message type)
- Encrypted payload (no plaintext)

---

## 9. Threat Model & Limitations

### What ZenTalk Protects Against:
| Threat | Protection |
|--------|-----------|
| Relay operator reading messages | ✅ Double Ratchet E2EE |
| Relay operator reading old messages | ✅ Forward secrecy |
| Network eavesdropping | ✅ TLS + E2EE |
| Compromised relay server | ✅ Onion routing + E2EE |
| Database leak | ✅ Only encrypted blobs exposed |

### What ZenTalk DOES NOT Protect Against:
| Threat | Mitigation |
|--------|------------|
| Compromised end-user device | Use secure hardware |
| Malicious client app | Verify app signature |
| Traffic analysis (timing/size) | Use Tor or VPN |
| Quantum computers (future) | Post-quantum crypto upgrade needed |

---

## 10. Comparison with Other Protocols

| Feature | ZenTalk | Signal | WhatsApp | Telegram |
|---------|---------|--------|----------|----------|
| End-to-End Encryption | ✅ Yes | ✅ Yes | ✅ Yes | ⚠️ Optional |
| Forward Secrecy | ✅ Yes | ✅ Yes | ✅ Yes | ⚠️ Limited |
| Onion Routing | ✅ Yes | ❌ No | ❌ No | ❌ No |
| Decentralized | ✅ Yes | ❌ No | ❌ No | ❌ No |
| Server Cannot Read | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No (non-secret) |
| Metadata Privacy | ⚠️ Partial | ⚠️ Partial | ❌ No | ❌ No |
| Open Source | ✅ Yes | ✅ Yes | ❌ No | ⚠️ Clients only |

---

## 11. Recommendations for Relay Operators

### Storage Best Practices
1. **Encrypt database at rest**: Use full disk encryption
2. **Secure key storage**: Protect relay private key with HSM
3. **Limit log retention**: Don't log message metadata
4. **Regular cleanup**: Enable automatic TTL expiration
5. **Access control**: Restrict database access

### Privacy Enhancements
1. **Don't log IP addresses**: Minimize connection logs
2. **Use Tor hidden service**: Accept Tor connections
3. **Rate limiting**: Prevent traffic analysis
4. **Random delays**: Add jitter to forwarding timing

### Security Monitoring
1. **Monitor queue size**: Detect DoS attacks
2. **Track delivery success**: Identify network issues
3. **Alert on anomalies**: Unusual traffic patterns
4. **Regular key rotation**: Rotate relay keys periodically

---

## 12. Technical Details

### Message Flow with Encryption

```
┌──────────────────────────────────────────────────────────────┐
│ 1. SENDER'S DEVICE                                           │
├──────────────────────────────────────────────────────────────┤
│ • Create plaintext message: "Hello!"                         │
│ • Look up recipient's key bundle from DHT                    │
│ • Perform X3DH key agreement → shared secret                 │
│ • Initialize Double Ratchet session                          │
│ • Encrypt message with ratchet:                              │
│   - Derive message key from chain key                        │
│   - AES-256-GCM encrypt: ciphertext = E(key, plaintext)      │
│   - Create header: DH pub key + message number               │
│ • Build onion layers:                                        │
│   - Layer 3: RSA-4096 encrypt for Relay 3                    │
│   - Layer 2: RSA-4096 encrypt for Relay 2                    │
│   - Layer 1: RSA-4096 encrypt for Relay 1                    │
│ • Send to Relay 1                                            │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 2. RELAY 1 (First Hop)                                       │
├──────────────────────────────────────────────────────────────┤
│ • Receives encrypted onion packet                            │
│ • Decrypt Layer 1 with private key:                          │
│   - Reveals: next_hop = Relay 2                              │
│   - Reveals: remaining_payload (still encrypted)             │
│ • CANNOT see:                                                │
│   - Final recipient ❌                                       │
│   - Message content ❌                                       │
│   - Sender identity ❌                                       │
│ • Forward remaining_payload to Relay 2                       │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 3. RELAY 2 (Middle Hop)                                      │
├──────────────────────────────────────────────────────────────┤
│ • Decrypt Layer 2 with private key:                          │
│   - Reveals: next_hop = Relay 3                              │
│   - Reveals: remaining_payload (still encrypted)             │
│ • Forward to Relay 3                                         │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 4. RELAY 3 (Exit/Delivery Relay)                             │
├──────────────────────────────────────────────────────────────┤
│ • Decrypt Layer 3 with private key:                          │
│   - Reveals: next_hop = Recipient Address                    │
│   - Reveals: ratchet_payload (header + ciphertext)           │
│ • CANNOT see:                                                │
│   - Message content ❌ (Double Ratchet encrypted)            │
│ • Check if recipient is online:                              │
│   - If ONLINE: forward immediately                           │
│   - If OFFLINE: queue in database (encrypted blob)           │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 5. RECIPIENT'S DEVICE                                        │
├──────────────────────────────────────────────────────────────┤
│ • Receives ratchet payload (header + ciphertext)             │
│ • Decode ratchet header:                                     │
│   - Extract: sender's DH public key                          │
│   - Extract: message number                                  │
│ • Check if DH ratchet step needed                            │
│ • Derive message key from receiving chain                    │
│ • AES-256-GCM decrypt: plaintext = D(key, ciphertext)        │
│ • Advance ratchet state (new keys for next message)          │
│ • Display: "Hello!"                                          │
└──────────────────────────────────────────────────────────────┘
```

### Offline Message Queue Storage

**When recipient is OFFLINE**:

```sql
-- Relay stores this in database
INSERT INTO queued_messages (
    recipient_addr,      -- "efeaa8e5efdcb380bf8581944cd738f448b8a244"
    message_id,          -- "a1b2c3d4e5f6..."
    encrypted_payload,   -- [ENCRYPTED BLOB - UNREADABLE]
    timestamp,           -- 1705320000
    expires_at,          -- 1707912000 (30 days later)
    attempts             -- 0
);
```

**When recipient comes ONLINE**:
```go
// Relay delivers all queued messages
messages := relay.GetQueuedMessages(recipientAddr)
for _, msg := range messages {
    relay.DeliverMessage(recipientAddr, msg.EncryptedPayload)
    relay.DeleteMessage(msg.MessageID) // Remove from queue
}
```

**Relay operator inspects database**:
```bash
$ sqlite3 relay-9001-queue.db
sqlite> SELECT * FROM queued_messages;

# RESULT:
recipient_addr: efeaa8e5efdcb380bf8581944cd738f448b8a244
message_id: a1b2c3d4e5f6...
encrypted_payload: [BINARY BLOB OF RANDOM-LOOKING DATA]
                   # Relay CANNOT decrypt this!
                   # Contains:
                   #   - Double Ratchet header (40 bytes)
                   #   - AES-256-GCM ciphertext
                   #   - Authentication tag
```

---

## 13. Cryptographic Primitives

| Component | Algorithm | Key Size | Purpose |
|-----------|-----------|----------|---------|
| Message Encryption | AES-256-GCM | 256-bit | Symmetric encryption |
| Key Exchange | X25519 (Curve25519) | 256-bit | ECDH for ratchet |
| Key Derivation | HKDF-SHA256 | 256-bit | Derive keys from secrets |
| Onion Routing | RSA-OAEP | 4096-bit | Per-hop encryption |
| Digital Signatures | Ed25519 | 256-bit | Identity verification |
| Random Generation | crypto/rand | - | Cryptographically secure |

---

## 14. Conclusion

### Privacy Summary

✅ **Messages are truly private**:
- Relay operators **cannot** read message content
- Relay operators **cannot** decrypt old messages (forward secrecy)
- Relay operators **cannot** impersonate users
- Relay operators **cannot** modify messages (authenticated encryption)

⚠️ **Metadata visibility**:
- Relays **can** see recipient addresses (needed for routing)
- Relays **can** see message sizes and timing
- Relays **can** perform traffic analysis
- Use Tor/VPN for IP privacy

### Message Deletion Reality

When you delete a message:
- ✅ Your device: Message deleted
- ✅ Your API server: Message deleted
- ⚠️ Relay servers: Encrypted blob may still exist (unreadable)
- ❌ Recipient's device: Message still exists
- ❌ Other participants: Message still exists

**This is by design**: True E2EE means you don't control others' copies.

### Trust Model

You must trust:
- ✅ Your own device security
- ✅ Recipient's device security
- ✅ Cryptographic algorithms

You do NOT need to trust:
- ❌ Relay operators (E2EE protects content)
- ❌ Network providers (TLS + E2EE)
- ❌ DHT nodes (Public key bundles are signed)

---

**For more information**:
- Protocol Specification: `/docs/PROTOCOL.md`
- Threat Model: `/docs/THREAT_MODEL.md`
- Security Audit: Contact security@zentalk.io

**Report vulnerabilities**: security@zentalk.io (PGP key available)
