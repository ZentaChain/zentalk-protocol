# ZenTalk Protocol - Architecture Documentation

## Table of Contents

- [Overview](#overview)
- [Package Structure](#package-structure)
- [Detailed Package Explanations](#detailed-package-explanations)
  - [1. pkg/api - REST API & WebSocket Server](#1-pkgapi---rest-api--websocket-server)
  - [2. pkg/crypto - Cryptographic Primitives](#2-pkgcrypto---cryptographic-primitives)
  - [3. pkg/dht - Distributed Hash Table (Kademlia)](#3-pkgdht---distributed-hash-table-kademlia)
  - [4. pkg/network - Network Layer](#4-pkgnetwork---network-layer)
  - [5. pkg/protocol - Protocol Definitions](#5-pkgprotocol---protocol-definitions)
  - [6. pkg/storage - Data Persistence](#6-pkgstorage---data-persistence)
- [System Flows](#system-flows)
  - [Message Sending Flow](#message-sending-flow-end-to-end)
  - [Relay Discovery Flow](#relay-discovery-flow)
  - [Encryption Layers](#encryption-layers)

---

## Overview

ZenTalk is a decentralized, privacy-focused messaging protocol that combines:
- **Signal Protocol** (X3DH + Double Ratchet) for end-to-end encryption
- **Tor-style Onion Routing** for metadata privacy
- **Kademlia DHT** for decentralized peer discovery
- **MeshStorage** for distributed, encrypted media storage

The system provides Signal-level encryption with Tor-level anonymity in a fully decentralized architecture.

---

## Package Structure

```
pkg/
├── api/          (23 files) - REST API & WebSocket server
├── crypto/       (3 files)  - Cryptographic primitives
├── dht/          (6 files)  - Distributed Hash Table (Kademlia)
├── network/      (21 files) - Network layer & relay servers
├── protocol/     (9 files)  - Protocol definitions & encryption
└── storage/      (7 files)  - Data persistence & queuing
```

---

## Detailed Package Explanations

### 1. pkg/api - REST API & WebSocket Server

**Purpose:** HTTP/WebSocket interface for clients (web, mobile apps)

**What it does:**
- Provides REST endpoints for account creation, login, messaging
- Real-time WebSocket connections for instant message delivery
- Multi-tenant session management (multiple users per server)
- Handles message encryption/decryption for offline storage
- Manages read receipts, typing indicators, starred messages
- Media upload/download via MeshStorage (encrypted, distributed)

**Key Components:**
- `server.go` - Starts HTTP server, registers routes
- `websocket.go` - Manages WebSocket connections for real-time updates
- `session.go` - Tracks active user sessions (username → network client mapping)
- `messages.go` - Send/receive message endpoints
- `contacts.go` - Contact management endpoints
- `database_*.go` - SQLite database operations for persistent storage

**How clients use it:**
```
Web/Mobile App → HTTP REST API → API Server → Network Layer → Protocol Relay Network
```

**Key Endpoints:**
- `POST /api/account/create` - Create new account
- `POST /api/account/login` - Login to existing account
- `POST /api/messages/send` - Send encrypted message
- `GET /api/messages/:conversationId` - Retrieve conversation history
- `POST /api/contacts/add` - Add contact
- `WS /api/ws` - WebSocket connection for real-time updates

---

### 2. pkg/crypto - Cryptographic Primitives

**Purpose:** Low-level cryptographic operations

**What it does:**
- **Hashing:** BLAKE2b-256 for data integrity, address generation
- **Asymmetric Encryption:** RSA-4096 for key exchange and signatures
- **Onion Routing:** Multi-layer encryption like Tor for anonymous message routing

**Key Functions:**
```go
// Hashing
Hash(data []byte) ([]byte, error)                    // BLAKE2b-256 hash
HashString(data []byte) (string, error)              // Hash as hex string
GenerateNonce(size int) ([]byte, error)              // Random nonce

// RSA Operations
GenerateRSAKeyPair() (*rsa.PrivateKey, error)        // Create RSA-4096 keys
RSAEncrypt(data, pubKey) ([]byte, error)             // Public key encryption
RSADecrypt(ciphertext, privKey) ([]byte, error)      // Private key decryption
SignData(data, privKey) ([]byte, error)              // Digital signature
VerifySignature(data, sig, pubKey) error             // Signature verification

// Onion Routing
CreateOnionPacket(payload, relays) ([]byte, error)   // Multi-layer encryption
DecryptOnionLayer(packet, privKey) (*OnionLayer, error) // Unwrap one layer
```

**Example Onion Routing Flow:**
```
Message → Encrypt with Relay3 key → Encrypt with Relay2 key → Encrypt with Relay1 key

Relay1 → Decrypt layer → Forward to Relay2
Relay2 → Decrypt layer → Forward to Relay3
Relay3 → Decrypt layer → Deliver to recipient
```

**Files:**
- `hash.go` (58 lines) - BLAKE2b-256 hashing utilities
- `keys.go` (137 lines) - RSA-4096 key management & operations
- `onion.go` (213 lines) - Tor-style onion routing implementation

---

### 3. pkg/dht - Distributed Hash Table (Kademlia)

**Purpose:** Peer discovery without central servers

**What it does:**
- Maintains distributed routing table of known peers
- Finds relay servers closest to target addresses
- Stores and retrieves peer information across the network
- Automatically discovers new relays through network crawling

**Key Concepts:**
- **XOR Metric:** Distance = XOR of node IDs (160-bit)
- **K-Buckets:** Store up to K=20 contacts per bucket (160 buckets total)
- **Iterative Lookup:** Query α=3 closest nodes, repeat until target found

**DHT Operations:**
- `PING` - Check if node is alive
- `STORE` - Store key-value pair
- `FIND_NODE` - Find closest nodes to target
- `FIND_VALUE` - Retrieve stored value

**Example Usage:**
```go
// Initialize DHT node
node := dht.NewNode(nodeID, address, privateKey)
node.Start()

// Bootstrap from known node
node.Bootstrap(bootstrapContact)

// Find relay servers for routing
relay := dht.FindRelay(targetAddress)

// Publish relay metadata
dht.Store(relayAddress, relayMetadata)
```

**Files:**
- `node_id.go` (133 lines) - 160-bit Kademlia node identifiers
- `routing_table.go` (234 lines) - K-bucket routing table implementation
- `protocol.go` (335 lines) - Core Kademlia protocol (store, lookup, bootstrap)
- `node.go` (269 lines) - DHT node lifecycle management
- `rpc.go` (129 lines) - RPC message types & serialization
- `storage.go` (107 lines) - Local key-value storage with TTL

---

### 4. pkg/network - Network Layer

**Purpose:** Manages connections, routing, message delivery

**What it does:**
- **Client:** Connects to relay servers, sends/receives messages
- **Relay Server:** Routes messages between clients and other relays
- **Session Management:** Tracks active user sessions (X3DH, Double Ratchet)
- **Message Routing:** Onion routing through multiple hops
- **Group Management:** Handles group messaging
- **Profile Management:** Stores and distributes user profiles

**Key Components:**

#### Client Side:
- `client.go` - Network client that connects to relays
- `message_sender.go` - Sends encrypted messages through onion routing
- `message_handler.go` - Processes incoming messages
- `x3dh_manager.go` - Manages X3DH key agreement
- `session_manager.go` - Manages Double Ratchet sessions
- `reconnect.go` - Auto-reconnect on connection loss

#### Relay Server Side:
- `relay.go` - Main relay server struct and lifecycle
- `relay_connection.go` - Accept and handle connections
- `relay_handlers.go` - Process protocol messages (handshake, ping, forward)
- `relay_delivery.go` - Message forwarding and delivery
- `relay_discovery.go` - DHT integration for relay discovery
- `relay_mesh.go` - Relay-to-relay connections (mesh network)
- `relay_registry.go` - Track known relays
- `relay_metadata.go` - Relay metadata structure

#### Supporting Components:
- `session_storage.go` - Persistent session storage
- `group_manager.go` - Group messaging logic
- `profile_manager.go` - User profile management
- `typing_receipt.go` - Typing indicators
- `pool.go` - Connection pooling
- `dht_integration.go` - DHT integration for relay discovery

**Message Routing Flow:**
```
Sender → Client.SendMessage()
→ Create onion packet (3 layers)
→ Connect to Relay1
→ Relay1 decrypts → forwards to Relay2
→ Relay2 decrypts → forwards to Relay3
→ Relay3 decrypts → delivers to Recipient
```

**Files Overview:**
- Client: 270 lines
- Relay Server: 323 lines (core) + 4 supporting files (~485 lines)
- Message Handling: 554 lines (handler) + 341 lines (sender)
- Session Management: 189 lines (manager) + 237 lines (storage)

---

### 5. pkg/protocol - Protocol Definitions

**Purpose:** Defines message formats and encryption protocols

**What it does:**
- **Wire Protocol:** Binary message format with headers
- **X3DH Protocol:** Extended Triple Diffie-Hellman key agreement (Signal)
- **Double Ratchet:** Forward secrecy encryption (Signal)
- **Message Types:** Handshake, Direct Message, Relay Forward, Group Message, etc.
- **Group Messaging:** Multi-party encrypted communication

**Key Message Types:**
```go
const (
    MsgTypeHandshake      = 0x0001  // Initial connection
    MsgTypeHandshakeAck   = 0x0002  // Handshake response
    MsgTypeDirectMessage  = 0x0003  // 1-on-1 message
    MsgTypeGroupMessage   = 0x0004  // Group message
    MsgTypeAck            = 0x0005  // Acknowledgment
    MsgTypeRelayForward   = 0x0006  // Onion-routed message
    MsgTypePing           = 0x0007  // Keep-alive
    MsgTypePong           = 0x0008  // Ping response
    MsgTypeProfileUpdate  = 0x000A  // Profile update
    MsgTypePresence       = 0x000B  // Presence notification
)
```

**Protocol Header:**
```go
type Header struct {
    Magic     uint32      // Protocol magic (0xDEADBEEF)
    Version   uint16      // Protocol version
    Type      uint16      // Message type
    Length    uint32      // Payload length
    Flags     uint32      // Message flags
    MessageID MessageID   // Unique message ID (16 bytes)
}
```

**Encryption Protocols:**

#### X3DH (Key Agreement):
```
Alice                          Bob
------                          ----
Identity Key (IK_A)            Identity Key (IK_B)
Signed Prekey (SPK_A)          Signed Prekey (SPK_B)
                               Onetime Prekeys (OPK_B)

Alice sends initial message:
- Ephemeral Key (EK_A)
- Uses Bob's IK_B, SPK_B, OPK_B
- Derives shared secret via 4 DH operations:
  DH1 = DH(IK_A, SPK_B)
  DH2 = DH(EK_A, IK_B)
  DH3 = DH(EK_A, SPK_B)
  DH4 = DH(EK_A, OPK_B)
  SK = KDF(DH1 || DH2 || DH3 || DH4)
- Initializes Double Ratchet with SK
```

#### Double Ratchet (Message Encryption):
```
Each message uses a new encryption key

Keys derived from:
- DH Ratchet (new key pair each exchange)
- Symmetric Ratchet (KDF chain)

Provides:
- Forward Secrecy (old keys deleted after use)
- Future Secrecy (compromise doesn't affect future messages)
```

**Files:**
- `x3dh.go` (548 lines) - X3DH key agreement implementation
- `ratchet.go` (416 lines) - Double Ratchet encryption
- `message.go` (418 lines) - Message structures & encoding
- `group.go` (279 lines) - Group messaging protocol
- `header.go` (111 lines) - Protocol header definitions
- `types.go` (121 lines) - Common type definitions
- `routing.go` (134 lines) - Message routing structures
- `profile.go` (93 lines) - User profile protocol
- `presence.go` (83 lines) - Presence notifications

---

### 6. pkg/storage - Data Persistence

**Purpose:** Encrypted local storage for messages and contacts

**What it does:**
- SQLite database with password-derived encryption keys
- Stores messages, contacts, conversations
- Offline message queue for relay servers
- Stores references to MeshStorage chunks (chunk IDs and encryption keys)

**Key Components:**
- `database.go` - Core initialization, schema, encryption
- `messages.go` - Save/retrieve/search messages
- `contacts.go` - Contact management (add, block, favorite)
- `conversations.go` - Conversation threads, unread counts
- `helpers.go` - Utility functions
- `relay_queue.go` - Offline message queue for relays

**Database Schema:**

```sql
-- Messages table
CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    conversation_id TEXT NOT NULL,
    message_id TEXT UNIQUE NOT NULL,
    from_address TEXT NOT NULL,
    to_address TEXT NOT NULL,
    content BLOB NOT NULL,              -- Encrypted message content
    content_type INTEGER NOT NULL,
    timestamp INTEGER NOT NULL,
    status TEXT NOT NULL,               -- sending, sent, delivered, read, failed
    is_outgoing INTEGER NOT NULL,
    media_chunk_id INTEGER,             -- MeshStorage chunk ID for media
    media_key BLOB,                     -- Encryption key for media
    reply_to_id TEXT,
    created_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now'))
);

-- Contacts table
CREATE TABLE contacts (
    address TEXT PRIMARY KEY,           -- User's ZenTalk address (BLAKE2b hash)
    username TEXT NOT NULL,
    bio TEXT,
    avatar_chunk_id INTEGER,            -- MeshStorage chunk ID for avatar
    avatar_key BLOB,                    -- Encryption key for avatar
    public_key BLOB,                    -- RSA public key
    added_at INTEGER NOT NULL,
    last_seen INTEGER,
    is_blocked INTEGER NOT NULL DEFAULT 0,
    is_favorite INTEGER NOT NULL DEFAULT 0
);

-- Conversations table
CREATE TABLE conversations (
    id TEXT PRIMARY KEY,                -- Conversation ID
    contact_address TEXT NOT NULL,
    last_message_id TEXT,
    last_message TEXT,
    last_timestamp INTEGER,
    unread_count INTEGER NOT NULL DEFAULT 0,
    is_muted INTEGER NOT NULL DEFAULT 0,
    is_pinned INTEGER NOT NULL DEFAULT 0,
    FOREIGN KEY (contact_address) REFERENCES contacts(address)
);
```

**Example Usage:**
```go
// Open encrypted database
db, err := storage.NewMessageDB("/path/to/db.sqlite", "password")

// Save message
msg := &storage.StoredMessage{
    ConversationID: convID,
    MessageID:      msgID,
    FromAddress:    senderAddr,
    ToAddress:      recipientAddr,
    Content:        encryptedContent,
    Timestamp:      time.Now().Unix(),
    Status:         storage.MessageStatusSent,
}
err = db.SaveMessage(msg)

// Get conversation messages
messages, err := db.GetConversationMessages(convID, 50, 0)
```

**Files:**
- `database.go` (182 lines) - Core database initialization
- `messages.go` (277 lines) - Message CRUD operations
- `contacts.go` (175 lines) - Contact management
- `conversations.go` (93 lines) - Conversation management
- `helpers.go` (70 lines) - Utility functions
- `relay_queue.go` (292 lines) - Offline message queue

---

## System Flows

### Message Sending Flow (End-to-End)

```
1. USER ACTION (Web/Mobile App)
   User types: "Hello Bob"
   ↓
2. API SERVER (pkg/api)
   POST /api/messages/send
   - Validates request
   - Looks up recipient's address
   ↓
3. SESSION MANAGER (pkg/network/session_manager.go)
   - Gets or creates Double Ratchet session with Bob
   - Encrypts message with session key
   ↓
4. MESSAGE SENDER (pkg/network/message_sender.go)
   - Selects 3 random relays from DHT
   - Creates onion packet
   ↓
5. ONION ROUTING (pkg/crypto/onion.go)
   - Layer 1: Encrypt(message, relay3_pubkey) → Payload1
   - Layer 2: Encrypt(Payload1, relay2_pubkey) → Payload2
   - Layer 3: Encrypt(Payload2, relay1_pubkey) → Payload3
   ↓
6. CLIENT (pkg/network/client.go)
   - Connects to Relay1
   - Sends: Header(MsgTypeRelayForward) + Payload3
   ↓
7. RELAY1 (pkg/network/relay_handlers.go)
   - Decrypts Payload3 with private key
   - Gets: NextHop=Relay2, Payload=Payload2
   - Forwards to Relay2
   ↓
8. RELAY2 (pkg/network/relay_handlers.go)
   - Decrypts Payload2 with private key
   - Gets: NextHop=Relay3, Payload=Payload1
   - Forwards to Relay3
   ↓
9. RELAY3 (pkg/network/relay_handlers.go)
   - Decrypts Payload1 with private key
   - Gets: NextHop=Bob's Address, Payload=Encrypted Message
   - Checks if Bob is connected
   ↓
10. RELAY3 → BOB'S CLIENT (pkg/network/relay_delivery.go)
    - If Bob online: Delivers immediately
    - If Bob offline: Queues in relay message queue
    ↓
11. BOB'S CLIENT (pkg/network/message_handler.go)
    - Receives encrypted message
    - Decrypts with Double Ratchet session
    - Verifies signature
    ↓
12. STORAGE (pkg/storage/messages.go)
    - Saves decrypted message to SQLite database
    - Updates conversation last_message
    ↓
13. WEBSOCKET (pkg/api/websocket.go)
    - Pushes real-time notification to Bob's web/mobile app
    - App displays: "Alice: Hello Bob"
```

**Metadata Privacy:**
- Relay1 knows: Alice's IP, Relay2's address
- Relay2 knows: Relay1's address, Relay3's address
- Relay3 knows: Relay2's address, Bob's address
- **No single relay knows both sender and recipient**

---

### Relay Discovery Flow

```
1. RELAY SERVER STARTS (pkg/network/relay.go)
   relay := NewRelayServer(port, privateKey)
   relay.Start()
   ↓
2. ATTACH DHT (pkg/network/relay.go)
   dhtNode := dht.NewNode(...)
   relay.AttachDHT(dhtNode)
   ↓
3. SET METADATA (pkg/network/relay.go)
   relay.SetRelayMetadata(
       region: "us-east",
       operator: "0x1234...",
       version: "1.0.0",
       maxConnections: 10000,
   )
   ↓
4. PUBLISH TO DHT (pkg/dht/protocol.go)
   relay.PublishToDHT()
   - Stores relay info at key = Hash(relay_address)
   - TTL: 1 hour (republished every 30 minutes)
   ↓
5. AUTO-REPUBLISH (background goroutine)
   relay.AutoPublishToDHT(30 * time.Minute)

---

CLIENT SIDE: Finding a Relay

6. CLIENT NEEDS RELAY (pkg/network/dht_integration.go)
   client.FindRelayForRecipient(recipientAddress)
   ↓
7. DHT LOOKUP (pkg/dht/protocol.go)
   dht.FindRelay(recipientAddress)
   - Calculates XOR distance to recipient
   - Queries α=3 closest nodes
   - Iteratively narrows search
   - Returns relay closest to recipient
   ↓
8. CLIENT CONNECTS (pkg/network/client.go)
   client.ConnectToRelay(relayAddress)
   - Opens TCP connection
   - Sends handshake
   - Stores connection in pool
```

**DHT Lookup Example:**
```
Target: Bob's Address = 0xABCD...
Query closest nodes:
- Node1 (distance: 0x5234...) → Returns closer nodes
- Node2 (distance: 0x3123...) → Returns closer nodes
- Node3 (distance: 0x1A2B...) → Returns relay at 0x0123...

Final Result: Relay at 0x0123... (closest to 0xABCD...)
```

---

### Encryption Layers

Visual representation of the encryption stack:

```
┌─────────────────────────────────────────────────┐
│ Application Layer                               │
│ - User types "Hello Bob"                        │
│ - Plaintext: "Hello Bob"                        │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ Double Ratchet Encryption                       │
│ (pkg/protocol/ratchet.go)                       │
│                                                  │
│ - Forward secrecy, new key per message          │
│ - Encrypted: E1("Hello Bob", ratchet_key)       │
│ - Output: 0x9A3F... (encrypted message)         │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ Onion Layer 1 (Final Hop - Relay3)             │
│ (pkg/crypto/onion.go)                           │
│                                                  │
│ - Next Hop: Bob's Address                       │
│ - Payload: 0x9A3F... (Double Ratchet output)    │
│ - Encrypted: E2(payload, relay3_pubkey)         │
│ - Output: 0x7B2E...                             │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ Onion Layer 2 (Middle Hop - Relay2)            │
│ (pkg/crypto/onion.go)                           │
│                                                  │
│ - Next Hop: Relay3's Address                    │
│ - Payload: 0x7B2E... (Layer 1 output)           │
│ - Encrypted: E3(payload, relay2_pubkey)         │
│ - Output: 0x4C1D...                             │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ Onion Layer 3 (First Hop - Relay1)             │
│ (pkg/crypto/onion.go)                           │
│                                                  │
│ - Next Hop: Relay2's Address                    │
│ - Payload: 0x4C1D... (Layer 2 output)           │
│ - Encrypted: E4(payload, relay1_pubkey)         │
│ - Output: 0x2F8A... (final packet)              │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ Protocol Layer                                  │
│ (pkg/protocol/header.go)                        │
│                                                  │
│ - Header: Magic, Version, Type, Length          │
│ - Message Type: MsgTypeRelayForward (0x0006)    │
│ - Payload: 0x2F8A...                            │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ Network Transmission                            │
│ (pkg/network/client.go)                         │
│                                                  │
│ - TCP connection to Relay1                      │
│ - Binary data over wire                         │
└─────────────────────────────────────────────────┘
```

**Decryption Process (at relays):**
```
Relay1:
- Receives: Header + 0x2F8A...
- Decrypts with private key
- Extracts: NextHop=Relay2, Payload=0x4C1D...
- Forwards to Relay2

Relay2:
- Receives: Header + 0x4C1D...
- Decrypts with private key
- Extracts: NextHop=Relay3, Payload=0x7B2E...
- Forwards to Relay3

Relay3:
- Receives: Header + 0x7B2E...
- Decrypts with private key
- Extracts: NextHop=Bob's Address, Payload=0x9A3F... (Double Ratchet encrypted)
- Delivers to Bob's client

Bob's Client:
- Receives: Header + 0x9A3F...
- Decrypts with Double Ratchet session
- Extracts: "Hello Bob" (plaintext)
```

---

## Security Properties

### End-to-End Encryption
- **X3DH:** Establishes shared secret without prior communication
- **Double Ratchet:** Provides forward secrecy and future secrecy
- **Only sender and recipient can decrypt message content**

### Metadata Privacy
- **Onion Routing:** No single relay knows both sender and recipient
- **DHT Discovery:** Relay selection is decentralized
- **No central server knows who is talking to whom**

### Cryptographic Algorithms
- **Hashing:** BLAKE2b-256
- **Asymmetric:** RSA-4096 (key exchange, signatures)
- **Symmetric:** AES-256-GCM (data encryption in Double Ratchet)
- **Key Derivation:** HKDF-SHA256

---

## Performance Characteristics

### Latency
- **Direct Connection:** ~50-100ms (single relay)
- **Onion Routing:** ~150-300ms (3 relays)
- **DHT Lookup:** ~200-500ms (initial relay discovery)

### Throughput
- **Message Size:** ~10KB typical (text), up to 1MB (with media metadata)
- **Relay Capacity:** ~10,000 concurrent connections per relay
- **DHT Scalability:** Supports millions of nodes (O(log N) lookups)

### Storage
- **Local Database:** SQLite with WAL mode
- **Media Storage:** MeshStorage (distributed, encrypted, erasure-coded)
- **Session Storage:** ~5KB per active conversation

---

## Deployment Modes

### 1. Centralized Mode (Development)
```
Web App → API Server → Single Relay → Clients
```
- Single API server
- Single relay server
- No DHT (hardcoded relay)
- Useful for testing

### 2. Decentralized Mode (Production)
```
Web App → API Server → DHT → Multiple Relays → Clients
```
- Multiple API servers (horizontal scaling)
- Relay mesh network (100+ relays)
- DHT for relay discovery
- Full privacy guarantees

### 3. Hybrid Mode
```
Web App → Managed API Server → Public Relay Network
```
- Managed API servers for reliability
- Connect to public relay network
- Balance between ease-of-use and decentralization

---

## Future Enhancements

### Planned Features
- [ ] Multi-device sync (same account on multiple devices)
- [ ] Voice/Video calling (WebRTC over onion routing)
- [ ] File sharing with progress tracking
- [ ] Disappearing messages (auto-delete after time)
- [ ] Message reactions and thread replies
- [ ] Push notifications (via Firebase/APNs)

### Performance Optimizations
- [ ] Connection pooling for relay connections
- [ ] Message batching for efficiency
- [ ] DHT caching to reduce lookups
- [ ] Prekey bundle prefetching

### Security Enhancements
- [ ] Post-quantum cryptography (Kyber, Dilithium)
- [ ] Zero-knowledge proofs for metadata privacy
- [ ] Blockchain-based relay reputation system

---

## References

### Protocols
- [Signal Protocol (X3DH + Double Ratchet)](https://signal.org/docs/)
- [Kademlia DHT](https://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf)
- [Tor Onion Routing](https://www.torproject.org/about/history/)

### Cryptography
- [BLAKE2](https://www.blake2.net/)
- [RSA](https://en.wikipedia.org/wiki/RSA_(cryptosystem))
- [AES-GCM](https://en.wikipedia.org/wiki/Galois/Counter_Mode)

### Technologies
- [libp2p](https://libp2p.io/) - P2P networking stack
- [SQLite](https://www.sqlite.org/)
- [Go Programming Language](https://go.dev/)

---

## License

See LICENSE file for details.

---

## Contributing

Contributions are welcome! Please see CONTRIBUTING.md for guidelines.

---

**Last Updated:** 2025-01-18
**Protocol Version:** 1.0.0
