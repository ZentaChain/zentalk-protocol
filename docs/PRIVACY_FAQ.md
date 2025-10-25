# ZenTalk Privacy FAQ - Quick Answers

## Q1: How does the relay server handle storage?

**Answer**: Relay servers store **encrypted message blobs** in a SQLite database.

### Storage Details:
- **Location**: `./data/relay-{port}-queue.db`
- **TTL**: 30 days (auto-delete after expiration)
- **What's stored**: Only messages for offline recipients
- **Online delivery**: Messages delivered immediately, NOT stored

### Database Structure:
```sql
queued_messages table:
- recipient_addr    -- Wallet address (visible)
- message_id        -- Random ID (visible)
- encrypted_payload -- ENCRYPTED BLOB (unreadable)
- timestamp         -- When queued (visible)
- expires_at        -- Expiration time (visible)
```

**Key Point**: Relay stores encrypted blobs, NOT plaintext messages.

---

## Q2: Are messages encrypted on the server?

**YES** - Messages are **end-to-end encrypted** before reaching the relay.

### Encryption Flow:

```
Sender Device
    ↓
[Encrypt with Double Ratchet]    ← Signal Protocol, AES-256-GCM
    ↓                               (only sender & recipient have keys)
[Wrap in Onion Layers]           ← RSA-4096 per relay hop
    ↓
Relay Server Receives:
    ↓
[Decrypts onion layer]            ← Relay removes its routing layer
    ↓
[Sees: Encrypted payload]         ← CANNOT decrypt message content
    ↓
[Stores encrypted blob]           ← Still encrypted with Double Ratchet
    ↓
Delivers to recipient when online
```

### What Relay CANNOT Decrypt:
- ❌ Message content ("Hello!")
- ❌ Sender's identity
- ❌ Message type (text/image/video)
- ❌ Media encryption keys
- ❌ Any plaintext data

### What Relay CAN See:
- ✅ Recipient address (needed for delivery)
- ✅ Message size (bytes)
- ✅ Timestamp
- ✅ Number of queued messages

**Code Reference**: `pkg/storage/relay_queue.go:95`
```go
func (q *RelayMessageQueue) QueueMessage(recipientAddr protocol.Address,
    messageID [16]byte, encryptedPayload []byte) error {
    // Relay stores the encrypted payload as-is
    // Cannot decrypt because it doesn't have ratchet session keys
}
```

---

## Q3: What happens when people delete their messages?

**Answer**: Message deletion is **client-side only** - this is how E2EE works.

### Deletion Flow:

When you delete a message:

| Location | What Happens |
|----------|--------------|
| **Your device** | ✅ **DELETED** immediately |
| **Your API server** | ✅ **DELETED** from database |
| **Relay queue** | ⚠️ **MAY EXIST** (encrypted blob, unreadable) |
| **Recipient's device** | ❌ **NOT DELETED** (they own their copy) |
| **Other participants** | ❌ **NOT DELETED** (they own their copy) |

### Why Recipient Keeps the Message:

This is **by design** in end-to-end encrypted systems:
- You control YOUR device and YOUR copy
- Recipient controls THEIR device and THEIR copy
- No central server can delete both copies
- This is how Signal, WhatsApp, and iMessage work too

### Relay Queue Behavior:

If message is queued (recipient offline):
- Relay has **encrypted blob** (can't read it)
- **Auto-deletes** after 30 days (TTL)
- Even if relay keeps it forever, it's **permanently encrypted**

**Code Reference**: `pkg/api/database_messages.go:165`
```go
// Deletes message from YOUR local database
func (db *DB) DeleteMessage(userAddr, peerAddr, messageID string) error {
    query := `DELETE FROM messages
              WHERE user_address = ?
              AND peer_address = ?
              AND id = ?`
    _, err := db.Conn.Exec(query, userAddr, peerAddr, messageID)
    return err
    // Note: This only deletes from YOUR device
}
```

---

## Summary Table

| Component | Storage | Encryption | Can Read Content? | Deletion |
|-----------|---------|------------|-------------------|----------|
| **Your Device** | Plaintext | - | ✅ Yes (your device) | ✅ You can delete |
| **Relay Server** | Encrypted blob | Double Ratchet + Onion | ❌ No (E2EE) | ⚠️ Auto-delete 30 days |
| **Recipient Device** | Plaintext | - | ✅ Yes (their device) | ❌ You cannot delete |
| **Network (attacker)** | Encrypted traffic | TLS + E2EE | ❌ No | N/A |

---

## Real-World Example

**Scenario**: Alice sends "Hello" to Bob

### Message Journey:

```
1. Alice's Device
   ├─ Plaintext: "Hello"
   ├─ Encrypt with Double Ratchet (AES-256-GCM)
   └─ Result: 0x8f3a9b2c... (encrypted blob)

2. Relay Server Receives
   ├─ Sees: Encrypted blob (1247 bytes)
   ├─ Sees: Recipient = Bob's address
   ├─ Cannot see: "Hello" (encrypted)
   ├─ Cannot see: Alice's address (onion routing)
   └─ Action: Store encrypted blob if Bob offline

3. Relay Database
   ├─ recipient_addr: 0xefeaa8e5...
   ├─ message_id: a1b2c3d4...
   ├─ encrypted_payload: 0x8f3a9b2c... [ENCRYPTED]
   ├─ timestamp: 1705320000
   └─ expires_at: 1707912000 (30 days later)

4. Bob Comes Online
   ├─ Relay delivers: 0x8f3a9b2c... (still encrypted)
   ├─ Bob's device decrypts with ratchet session
   └─ Bob sees: "Hello"

5. Alice Deletes Message
   ├─ Alice's device: ✅ DELETED
   ├─ Relay server: ⚠️ Still has encrypted blob (can't read it)
   └─ Bob's device: ❌ Still has "Hello" (Bob's property)
```

---

## Privacy Guarantees

### What ZenTalk Guarantees:

✅ **Relay cannot read messages** (Double Ratchet E2EE)
✅ **Forward secrecy** (past messages safe even if keys leaked)
✅ **Sender anonymity** (from relay's perspective via onion routing)
✅ **Automatic cleanup** (queued messages expire after 30 days)

### What ZenTalk Cannot Guarantee:

⚠️ **Recipient cooperation** (they own their copy of messages)
⚠️ **Device security** (if your device is compromised, messages exposed)
⚠️ **Network metadata** (relay sees recipient address and timing)
⚠️ **Quantum future** (current crypto vulnerable to quantum computers)

---

## Comparison: ZenTalk vs Others

| Feature | ZenTalk | Signal | WhatsApp | Telegram |
|---------|---------|--------|----------|----------|
| Relay reads messages? | ❌ No | ❌ No | ❌ No | ❌ No (secret chats only) |
| Forward secrecy? | ✅ Yes | ✅ Yes | ✅ Yes | ⚠️ Optional |
| Onion routing? | ✅ Yes | ❌ No | ❌ No | ❌ No |
| Decentralized relays? | ✅ Yes | ❌ No | ❌ No | ❌ No |
| Delete from recipient? | ❌ No | ⚠️ If within time window | ⚠️ If within time window | ✅ Yes (cloud delete) |
| Open source protocol? | ✅ Yes | ✅ Yes | ⚠️ Partial | ⚠️ Client only |

**Note**: Telegram's "delete for everyone" works because messages are stored on Telegram's servers (NOT true E2EE). ZenTalk prioritizes E2EE over deletion control.

---

## Verification Scripts

### 1. Test Messaging (E2E):
```bash
cd /Users/harun/Desktop/zentalk-web2025/zentalk/src/zentalk-protocol
./test_messaging.sh
```

### 2. Verify Relay Cannot Decrypt:
```bash
./verify_relay_cannot_decrypt.sh
```

### 3. Inspect Relay Database (as relay operator):
```bash
sqlite3 ./data/relay-9001-queue.db

-- See what relay operator can see
SELECT
    recipient_addr,
    message_id,
    length(encrypted_payload) as size_bytes,
    datetime(timestamp, 'unixepoch') as queued_at
FROM queued_messages;

-- Try to view encrypted payload (you'll see random hex data)
SELECT hex(encrypted_payload) FROM queued_messages LIMIT 1;
```

---

## For Relay Operators

### Best Practices for Privacy:

1. **Encrypt database at rest** - Use full disk encryption (LUKS/BitLocker)
2. **Don't log metadata** - Minimize IP address and connection logging
3. **Enable TTL cleanup** - Already automatic, ensures old messages expire
4. **Secure your relay key** - Use HSM or secure key storage
5. **Monitor queue size** - Detect attacks without reading content
6. **Use Tor** - Accept connections via Tor hidden service

### What You Should Know:

- ✅ You store encrypted blobs for offline users
- ✅ You cannot decrypt message content
- ✅ You cannot modify messages (authenticated encryption)
- ✅ You can see metadata (recipient, size, timing)
- ⚠️ Users expect you to delete old messages (30 day TTL)
- ⚠️ Don't try to break encryption (illegal in many jurisdictions)

---

## Further Reading

- **SECURITY_ANALYSIS.md** - Deep technical analysis of ZenTalk encryption
- **test_messaging.sh** - Automated E2E encryption test
- **verify_relay_cannot_decrypt.sh** - Practical demonstration of E2EE

**Questions?** Open an issue or contact: security@zentalk.io
