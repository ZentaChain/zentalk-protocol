# Multi-Relay Architecture - Chat History & Message Routing

## The Confusion: Where is Chat History Stored?

### Two Types of Servers

```
┌─────────────────────────────────────────────────────────────┐
│ 1. RELAY SERVERS (port 9001)                                │
│    - Forward messages (like postal service)                 │
│    - Temporarily store encrypted blobs for offline users    │
│    - NO long-term chat history storage                      │
│    - You can connect to ANY relay                           │
│    - Relay operators CANNOT read messages                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 2. API SERVERS (port 3001)                                  │
│    - YOUR personal server (like your phone)                 │
│    - Stores YOUR chat history in plaintext                  │
│    - Your local SQLite database                             │
│    - Syncs with your device                                 │
│    - YOU control this server                                │
└─────────────────────────────────────────────────────────────┘
```

---

## Scenario: 100 Relay Servers

### Question: "User talks today on Server 1, tomorrow on Server 3 - where is chat history?"

**Answer**: Chat history is stored on YOUR API server (port 3001), NOT on relay servers!

### How It Works:

```
DAY 1: User connects to Relay 1
┌──────────────────────────────────────────────────────────────┐
│  User's Device/API Server (localhost:3001)                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Chat History Database (SQLite)                         │  │
│  │ - Conversation with Alice: [msg1, msg2, msg3]          │  │
│  │ - Conversation with Bob: [msg4, msg5]                  │  │
│  └────────────────────────────────────────────────────────┘  │
│                        ↓ Connect to relay                     │
│                   Relay 1 (9001)                              │
│                   - Forwards messages                         │
│                   - Does NOT store chat history              │
└──────────────────────────────────────────────────────────────┘

DAY 2: User connects to Relay 3
┌──────────────────────────────────────────────────────────────┐
│  SAME User's Device/API Server (localhost:3001)              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Chat History Database (SAME database)                  │  │
│  │ - Conversation with Alice: [msg1, msg2, msg3, msg6]    │  │
│  │ - Conversation with Bob: [msg4, msg5, msg7]            │  │
│  └────────────────────────────────────────────────────────┘  │
│                        ↓ Connect to relay                     │
│                   Relay 3 (9003)                              │
│                   - Forwards messages                         │
│                   - Does NOT store chat history              │
└──────────────────────────────────────────────────────────────┘

✅ Chat history persists because it's stored LOCALLY
✅ User can connect to any relay (they're just routers)
```

---

## What Relay Servers Store

### Relay Database (`./data/relay-9001-queue.db`)

**ONLY stores**:
- Encrypted message blobs for OFFLINE recipients
- Temporary storage (30 day TTL)
- Deleted after delivery

**Example**:
```sql
-- Relay Server Database
queued_messages:
  recipient_addr: 0xabc123...
  encrypted_payload: [ENCRYPTED BLOB - UNREADABLE]
  expires_at: 2025-02-15 (30 days)

-- This is NOT chat history!
-- This is just a delivery queue
```

### API Server Database (`./data/zentalk.db`)

**Stores**:
- YOUR complete chat history
- YOUR contacts
- YOUR account info
- Plaintext (because it's YOUR server)

**Example**:
```sql
-- API Server Database (YOUR database)
messages:
  user_address: 0x123...
  peer_address: 0xabc...
  content: "Hello!" (plaintext on YOUR server)
  timestamp: 2025-01-15

-- This is YOUR chat history
-- Persists forever (until you delete)
```

---

## Message Flow with Multiple Relays

### Scenario: Alice → Relay 1 → Relay 2 → Relay 3 → Bob

```
1. Alice sends message "Hello Bob"
   ├─ Encrypt with Double Ratchet
   ├─ Build onion layers (3 relays)
   └─ Send to Relay 1

2. Relay 1
   ├─ Decrypt Layer 1 (sees: next_hop = Relay 2)
   ├─ Forward to Relay 2
   └─ Does NOT store message (online routing)

3. Relay 2
   ├─ Decrypt Layer 2 (sees: next_hop = Relay 3)
   ├─ Forward to Relay 3
   └─ Does NOT store message

4. Relay 3 (Bob's relay)
   ├─ Decrypt Layer 3 (sees: recipient = Bob)
   ├─ Check if Bob is online
   └─ If ONLINE: deliver immediately
      If OFFLINE: store encrypted blob in queue

5. Bob receives message
   ├─ Decrypt with Double Ratchet
   ├─ Store in HIS API server database (plaintext)
   └─ Relay 3 deletes encrypted blob from queue

6. Alice's chat history (on her API server)
   ├─ Stores: "You: Hello Bob"
   └─ Persists even if she switches relays

7. Bob's chat history (on his API server)
   ├─ Stores: "Alice: Hello Bob"
   └─ Persists even if he switches relays
```

---

## Multi-Relay Network Architecture

```
              ┌─────────────┐
              │  Relay 1    │
              │  (9001)     │
              └──────┬──────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
    ┌────▼───┐  ┌───▼────┐  ┌───▼────┐
    │Relay 2 │  │Relay 3 │  │Relay 4 │
    │(9002)  │  │(9003)  │  │(9004)  │
    └────┬───┘  └───┬────┘  └───┬────┘
         │          │           │
         └──────────┼───────────┘
                    │
         ┌──────────▼──────────┐
         │                     │
    ┌────▼─────┐        ┌─────▼────┐
    │  Alice   │        │   Bob    │
    │ (API:    │        │ (API:    │
    │  3001)   │        │  3002)   │
    └──────────┘        └──────────┘
         │                     │
    ┌────▼──────────┐    ┌────▼──────────┐
    │ Alice's Chat  │    │ Bob's Chat    │
    │ History DB    │    │ History DB    │
    │ (SQLite)      │    │ (SQLite)      │
    └───────────────┘    └───────────────┘
```

**Key Points**:
- Relays are interconnected (mesh network)
- Users connect to ANY relay
- Chat history stored on user's API server
- Relays only forward messages

---

## Can 100 Relay Servers Delete/Decrypt Messages?

### Delete Messages?

**Relay Queue Messages**:
- ✅ Relay operators CAN delete encrypted blobs from their queue
- ⚠️ But message already delivered to recipient's API server
- ⚠️ Deleting from relay queue doesn't delete from chat history

**Chat History**:
- ❌ Relay operators CANNOT delete your chat history
- Your chat history is on YOUR API server, not relay servers

### Decrypt Messages?

**NO** - Relay operators cannot decrypt messages:
- Messages encrypted with Double Ratchet (E2EE)
- Only sender and recipient have session keys
- Relay only has onion routing key (already peeled)
- Even with 100 relay operators colluding, they can't decrypt

---

## Data Persistence: User Switches Devices

### Scenario: User has 2 devices (laptop + phone)

```
Option 1: Separate API Servers (Different Chat History)
┌──────────────────────────────────────────────────────┐
│ Laptop (API Server 1 - localhost:3001)              │
│ ├─ Chat history with Alice: [msg1, msg2, msg3]      │
│ ├─ Connects to Relay 1                              │
│ └─ SQLite: ./data/zentalk-laptop.db                 │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│ Phone (API Server 2 - phone:3001)                   │
│ ├─ Chat history with Alice: [msg4, msg5]            │
│ ├─ Connects to Relay 2                              │
│ └─ SQLite: ./data/zentalk-phone.db                  │
└──────────────────────────────────────────────────────┘

⚠️ Problem: Different chat histories on each device!
```

```
Option 2: Cloud Sync API Server (Same Chat History)
┌──────────────────────────────────────────────────────┐
│ Cloud API Server (cloud.example.com:3001)           │
│ ├─ Chat history with Alice: [msg1, msg2, msg3]      │
│ ├─ Synced across all devices                        │
│ └─ SQLite: ./data/zentalk-user123.db                │
└──────────────────────────────────────────────────────┘
         ↑                           ↑
    ┌────┴────┐                 ┌───┴────┐
    │ Laptop  │                 │ Phone  │
    │ (connects to relay 1)     │ (connects to relay 2)
    └─────────┘                 └────────┘

✅ Solution: Both devices connect to same cloud API server
✅ Chat history persists across devices
```

---

## Code References

### Relay Queue Storage (Temporary)

**File**: `pkg/storage/relay_queue.go:95`
```go
// Relay stores encrypted blob for offline recipient
func (q *RelayMessageQueue) QueueMessage(
    recipientAddr protocol.Address,
    messageID [16]byte,
    encryptedPayload []byte) error {

    // Store encrypted blob
    query := `INSERT INTO queued_messages
              (recipient_addr, message_id, encrypted_payload, ...)
              VALUES (?, ?, ?, ...)`

    // TTL: 30 days
    expiresAt := now + int64(q.ttl.Seconds())
}
```

### API Server Chat History (Permanent)

**File**: `pkg/api/database_messages.go:12`
```go
// API server stores YOUR chat history (plaintext on your server)
func (db *DB) SaveMessage(userAddr, peerAddr string, msg Message) error {
    query := `INSERT OR REPLACE INTO messages
              (id, user_address, peer_address, content, timestamp, ...)
              VALUES (?, ?, ?, ?, ?, ...)`

    // Stores forever (until you delete)
    _, err := db.Conn.Exec(query, msg.ID, userAddr, peerAddr, msg.Content, ...)
}
```

### Message Delivery Flow

**File**: `pkg/network/relay_handlers.go:56`
```go
// When recipient comes online, relay delivers queued messages
func (rs *RelayServer) handleHandshake(conn net.Conn, header *protocol.Header) {
    // Peer connected, check for queued messages
    if rs.messageQueue != nil && hs.ClientType == protocol.ClientTypeUser {
        go rs.deliverQueuedMessages(hs.Address)
        // After delivery, messages are DELETED from relay queue
    }
}
```

---

## Summary Table

| Aspect | Relay Server | API Server |
|--------|--------------|------------|
| **Purpose** | Message routing | Store YOUR data |
| **Port** | 9001 | 3001 |
| **Storage** | Temporary queue (30 days) | Permanent chat history |
| **What's stored** | Encrypted blobs (offline users) | YOUR messages (plaintext) |
| **Can decrypt?** | ❌ No (E2EE) | ✅ Yes (your server) |
| **Persistence** | Deleted after delivery | Forever (until you delete) |
| **Switchable?** | ✅ Connect to any relay | ⚠️ Need to sync across devices |
| **Who controls?** | Relay operator | YOU |

---

## Best Practice: Production Deployment

### For Users:

```
┌─────────────────────────────────────────────────────────┐
│ YOUR SETUP (Recommended)                                │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Run YOUR API Server (cloud or local)                │
│     - Stores YOUR chat history                          │
│     - Accessible from all your devices                  │
│     - YOU control the data                              │
│                                                          │
│  2. Connect to PUBLIC Relay Network                     │
│     - Use any available relay                           │
│     - Relay just forwards messages                      │
│     - Can switch relays anytime                         │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### For Relay Operators:

```
┌─────────────────────────────────────────────────────────┐
│ RELAY OPERATOR SETUP                                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Run Relay Server (port 9001)                        │
│     - Forward messages                                  │
│     - Queue for offline users (30 day TTL)              │
│     - NO long-term storage                              │
│                                                          │
│  2. Do NOT run API servers for users                    │
│     - Users run their own API servers                   │
│     - Relays don't store chat history                   │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Next: Create Multi-Relay Test

I'll create a test script that demonstrates:
1. Start 3 relay servers
2. User connects to Relay 1, sends message
3. User switches to Relay 3
4. Chat history persists (stored in API server, not relay)
5. Inspect relay databases (only temporary queue, no history)

See: `test_multi_relay.sh`
