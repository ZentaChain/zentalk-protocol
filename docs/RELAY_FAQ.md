# Relay Server FAQ - Simple Answers

## Your Questions Answered

### Q: "What is a relay? Why do they have a database?"

**Answer**: Relay servers are **message routers** (like a postal service). They have a database to queue messages for OFFLINE users only.

**Two types of servers**:
```
┌────────────────────────────────────┐
│ RELAY SERVER (port 9001)           │
│ - Routes messages                  │
│ - Queues for offline users (30d)  │
│ - Stores ENCRYPTED blobs           │
│ - NOT for chat history             │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│ API SERVER (port 3001)             │
│ - YOUR personal server             │
│ - Stores YOUR chat history         │
│ - Stores plaintext (your server)   │
│ - Persists forever                 │
└────────────────────────────────────┘
```

---

### Q: "When there are 100 servers, can they delete/decrypt messages?"

**Answer**:

**Delete**:
- ✅ Relay operators CAN delete encrypted blobs from THEIR queue
- ❌ But they CANNOT delete from YOUR API server (your chat history)
- ⚠️ Deleting relay queue doesn't help (messages already delivered)

**Decrypt**:
- ❌ NO - Relay operators CANNOT decrypt messages
- Messages are E2EE encrypted (Double Ratchet / Signal Protocol)
- Even with 100 relay operators colluding, they can't decrypt
- Only sender & recipient have the encryption keys

---

### Q: "How does chat history work when user talks on Server 1 today, Server 3 tomorrow?"

**Answer**: Chat history is stored on YOUR API server (port 3001), NOT on relay servers!

```
Today: User connects to Relay 1
┌──────────────────────────────────────┐
│  YOUR API Server (localhost:3001)   │
│  ┌────────────────────────────────┐ │
│  │ Chat History:                  │ │
│  │ - Alice: "Hi Bob"              │ │
│  │ - Bob: "Hello Alice"           │ │
│  │ - Alice: "How are you?"        │ │
│  └────────────────────────────────┘ │
│           ↓ connects to             │
│      Relay 1 (9001)                 │
│      - Just forwards messages       │
└──────────────────────────────────────┘

Tomorrow: User connects to Relay 3
┌──────────────────────────────────────┐
│  SAME API Server (localhost:3001)   │
│  ┌────────────────────────────────┐ │
│  │ Chat History: STILL THERE!     │ │
│  │ - Alice: "Hi Bob"              │ │
│  │ - Bob: "Hello Alice"           │ │
│  │ - Alice: "How are you?"        │ │
│  │ - Alice: "New message today"   │ │
│  └────────────────────────────────┘ │
│           ↓ connects to             │
│      Relay 3 (9003)                 │
│      - Just forwards messages       │
└──────────────────────────────────────┘

✅ Chat history persists!
✅ You can switch relays anytime
✅ Relay servers don't store chat history
```

**Key Points**:
1. Relays are just routers
2. Your chat history lives on YOUR API server
3. You can connect to ANY relay
4. Chat history always persists

---

## 100+ Relay Servers Scenario

### How It Works

```
User connects to Relay Network:

         ┌─────┐  ┌─────┐  ┌─────┐
         │Relay│  │Relay│  │Relay│
         │  1  │──│  2  │──│  3  │
         └─────┘  └─────┘  └─────┘
            │        │        │
         ┌──┴────────┴────────┴───┐
         │                        │
      ┌──┴──┐  ┌─────┐  ┌─────┐  │
      │Relay│  │Relay│  │Relay│  │
      │  4  │──│  5  │──│ ... │──│
      └─────┘  └─────┘  └─────┘  │
         │                        │
       ┌─┴────┐             ┌─────┴──┐
       │ YOU  │             │ Friend │
       │(API) │             │ (API)  │
       └──────┘             └────────┘
          │                      │
    ┌─────▼──────┐         ┌─────▼──────┐
    │ YOUR Chat  │         │ Friend's   │
    │ History    │         │ Chat       │
    │ Database   │         │ History    │
    └────────────┘         └────────────┘
```

**What happens**:
1. You connect to Relay 1 (or any relay)
2. Friend connects to Relay 50 (or any relay)
3. Message routes: You → Relay 1 → Relay 2 → ... → Relay 50 → Friend
4. Both store chat history on YOUR OWN API servers
5. Relays just forward, don't store chat history

---

## Demonstration Test Results

Run: `./test_multi_relay.sh`

**Results**:
```
✅ TEST RESULTS:

  1. Chat History Storage:
     API Server:   3 messages (permanent)
     Relay Server: 0 messages (temporary queue only)

  2. Encryption:
     Relay can read content: NO (E2EE encrypted)
     API server can read:    YES (your own server)

  3. Persistence:
     Switch relays: ✓ Chat history persists
     Switch devices: ⚠  Need to sync API server

  4. Privacy:
     Messages E2EE encrypted: ✓
     Relay cannot decrypt:    ✓
     Forward secrecy:         ✓
```

---

## Visual: Message Flow & Storage

### When Alice sends "Hello" to Bob:

```
1. Alice's Device
   ├─ Creates message: "Hello"
   ├─ Encrypts with Double Ratchet
   └─ Stores in Alice's API server: "You: Hello"

2. Send via Relay Network
   ├─ Relay 1: Forwards (encrypted blob)
   ├─ Relay 2: Forwards (encrypted blob)
   └─ Relay 3: Delivers to Bob (encrypted blob)

3. Bob's Device
   ├─ Receives encrypted blob
   ├─ Decrypts with Double Ratchet
   └─ Stores in Bob's API server: "Alice: Hello"

4. Relay Databases
   ├─ Relay 1 queue: 0 messages (online delivery)
   ├─ Relay 2 queue: 0 messages
   └─ Relay 3 queue: 0 messages

   (If Bob was offline, Relay 3 would store encrypted blob temporarily)
```

**Storage Summary**:
- Alice's API server: "You: Hello" (plaintext on her server)
- Bob's API server: "Alice: Hello" (plaintext on his server)
- Relay servers: Nothing (messages delivered immediately)

---

## Key Differences: Relay vs API Server

| Feature | Relay Server (9001) | API Server (3001) |
|---------|---------------------|-------------------|
| **Purpose** | Route messages | Store YOUR data |
| **Storage** | Temporary queue (offline users only) | Permanent chat history |
| **Duration** | 30 days TTL | Forever |
| **Format** | Encrypted blobs | Plaintext (your server) |
| **Can read?** | ❌ NO (E2EE) | ✅ YES (your server) |
| **Chat history?** | ❌ NO | ✅ YES |
| **Switchable?** | ✅ Connect to any relay | ⚠️ Need to sync across devices |
| **Who controls?** | Relay operator | YOU |

---

## Production Deployment

### For Users (Your Setup):

```
┌────────────────────────────────────────┐
│ 1. Run YOUR API Server                │
│    - Cloud: api.yourname.com:3001      │
│    - Local: localhost:3001             │
│    - Stores YOUR chat history          │
│                                        │
│ 2. Connect to Public Relay Network     │
│    - Use any available relay           │
│    - Relay just forwards messages      │
│    - Can switch relays anytime         │
└────────────────────────────────────────┘
```

### For Relay Operators:

```
┌────────────────────────────────────────┐
│ 1. Run Relay Server (port 9001)       │
│    - Forward messages                  │
│    - Queue for offline users (30d)     │
│    - Join mesh network                 │
│                                        │
│ 2. Do NOT store chat history          │
│    - Users run their own API servers   │
│    - Relays are just routers           │
└────────────────────────────────────────┘
```

---

## Summary

### Your Questions:

**Q: What is relay + why database?**
→ Message router + temporary queue for offline users

**Q: Can 100 servers delete/decrypt?**
→ Delete from queue: YES | Decrypt messages: NO (E2EE)

**Q: Chat history when switching servers?**
→ Stored in YOUR API server, NOT relay servers. Always persists.

### Key Takeaways:

1. **Relay servers** = Message routers (like postal service)
2. **API servers** = YOUR chat history database (like your phone)
3. **Switch relays** = No problem, chat history in API server
4. **100 relay operators** = Same architecture, just more routing
5. **Encryption** = Relay operators cannot decrypt (E2EE)
6. **Storage** = Relay queue (temporary) vs API server (permanent)

---

## Verify Yourself

```bash
# 1. Run the multi-relay test
./test_multi_relay.sh

# 2. Check API server (YOUR chat history)
curl -s http://localhost:3001/api/chats \
  -H "X-Wallet-Address: 0xaaa..." | jq '.chats'

# 3. Check relay queue (temporary storage)
sqlite3 ./data/relay-9001-queue.db \
  "SELECT count(*) FROM queued_messages;"

# Result: API server has chat history, relay queue is empty
```

---

## Related Documentation

- **MULTI_RELAY_ARCHITECTURE.md** - Full architecture explanation
- **SECURITY_ANALYSIS.md** - Encryption deep dive
- **PRIVACY_FAQ.md** - Privacy questions answered
- **test_multi_relay.sh** - Practical demonstration

---

**Questions?** Read MULTI_RELAY_ARCHITECTURE.md for complete details.
