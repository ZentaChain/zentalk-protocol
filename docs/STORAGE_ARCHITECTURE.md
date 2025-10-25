# ZenTalk Storage Architecture

This document explains the two different storage systems in ZenTalk and how they work together.

---

## Overview

ZenTalk uses **two complementary storage systems**:

1. **`pkg/storage`** - Local Message Database (SQLite)
2. **`pkg/meshstorage`** - Distributed P2P File Storage (Mesh Network)

Each system serves a distinct purpose and they work together to provide a complete messaging and file storage solution.

---

## 1. Local Message Database (`pkg/storage`)

### Purpose
Stores **local chat data** on the user's device using SQLite.

### What It Stores
- **Messages** - Sent and received chat messages
- **Contacts** - User contact list
- **Conversations** - Chat threads and metadata
- **Relay Queue** - Messages pending relay delivery
- **Media References** - Links to MeshStorage chunks (chunk IDs and encryption keys)

### Architecture
```
┌─────────────────────────────────────┐
│         User's Device               │
│                                     │
│  ┌──────────────────────────────┐  │
│  │   ZenTalk Client             │  │
│  │                              │  │
│  │  ┌────────────────────────┐  │  │
│  │  │   pkg/storage          │  │  │
│  │  │   (SQLite Database)    │  │  │
│  │  │                        │  │  │
│  │  │  • messages.db         │  │  │
│  │  │  • conversations.db    │  │  │
│  │  │  • contacts.db         │  │  │
│  │  │  • relay_queue.db      │  │  │
│  │  └────────────────────────┘  │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
```

### Key Files
- `database.go` - Core database initialization and management
- `messages.go` - Message CRUD operations
- `contacts.go` - Contact management
- `conversations.go` - Conversation tracking
- `relay_queue.go` - Message queue for relay delivery

### Use Cases
✅ **Chat messages** - Store and retrieve text messages
✅ **Contact list** - Manage user contacts
✅ **Message history** - Local message persistence
✅ **Offline support** - Store messages when offline
✅ **Quick access** - Fast local queries

### Characteristics
- **Storage Type:** SQLite (local file)
- **Location:** Single device only
- **Encryption:** No (relies on device encryption)
- **Redundancy:** None (device-dependent)
- **Access Speed:** Very fast (local disk)
- **Use Case:** Chat messages and metadata

---

## 2. Distributed Mesh Storage (`pkg/meshstorage`)

### Purpose
Provides **distributed, encrypted file storage** across a P2P mesh network with automatic redundancy.

### What It Stores
- **Files & Media** - Images, videos, documents, audio files
- **Encrypted Backups** - Encrypted user data backups
- **Large Attachments** - Files too large for direct messaging
- **Distributed Shards** - Erasure-coded file chunks

### Architecture
```
┌──────────────────────────────────────────────────────────────┐
│                    Mesh Storage Network                      │
│                                                              │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐   │
│  │  Node A    │◄────►│  Node B    │◄────►│  Node C    │   │
│  │  (Shards   │      │  (Shards   │      │  (Shards   │   │
│  │   1,5,9)   │      │   2,6,10)  │      │   3,7,11)  │   │
│  └────────────┘      └────────────┘      └────────────┘   │
│       ▲                    ▲                    ▲          │
│       │                    │                    │          │
│       └────────────────────┼────────────────────┘          │
│                            │                                │
│                    ┌───────▼────────┐                      │
│                    │   User Client  │                      │
│                    │   (Uploader)   │                      │
│                    └────────────────┘                      │
└──────────────────────────────────────────────────────────────┘

File Upload Process:
1. Encrypt file (AES-256-GCM)
2. Split into shards (Reed-Solomon 10+5)
3. Distribute shards across nodes
4. Store CID for retrieval
```

### Key Features

#### 🔐 Security
- **AES-256-GCM Encryption** - End-to-end encryption before distribution
- **Signature Verification** - RSA signatures for all operations
- **Privacy by Design** - Nodes cannot decrypt stored data
- **Secure Deletion** - Signature-verified shard removal

#### 🔄 Redundancy
- **Erasure Coding** - Reed-Solomon (10 data + 5 parity shards)
- **Automatic Repair** - Detects and repairs missing shards
- **Node Failure Tolerance** - Works with up to 5 failed nodes
- **Distributed Backup** - Data spread across multiple nodes

#### 🌐 P2P Network
- **DHT (Distributed Hash Table)** - Kademlia-based peer discovery
- **libp2p** - Industry-standard P2P networking
- **No Central Server** - Fully decentralized architecture
- **Bootstrap Nodes** - Optional initial peer discovery

#### 📦 Storage Management
- **Version Control** - Versioned storage format with migration
- **Encryption Key Rotation** - Support for key updates
- **Shard Health Monitoring** - Background health checks
- **Automatic Cleanup** - Expired shard removal

### Key Files
- `node.go` - DHT node and P2P networking
- `distributed.go` - Distributed storage coordination
- `storage.go` - Local shard storage
- `erasure.go` - Reed-Solomon erasure coding
- `encryption.go` - AES-256-GCM encryption
- `rpc.go` - RPC protocol for node communication
- `version.go` - Storage format versioning
- `migration.go` - Data migration system

### API Server (`pkg/meshstorage/api`)
REST API for interacting with mesh storage:
- `POST /upload` - Upload encrypted files
- `GET /download/:cid` - Download files by CID
- `GET /status/:cid` - Check file availability
- `DELETE /delete/:cid` - Delete files (signature required)
- `GET /network` - Network topology information

### Use Cases
✅ **Media sharing** - Photos, videos, voice messages
✅ **File attachments** - Documents, PDFs, archives
✅ **Encrypted backups** - Secure user data backups
✅ **Large content** - Files too big for direct messaging
✅ **Decentralized CDN** - Distributed content delivery

### Characteristics
- **Storage Type:** Distributed P2P mesh
- **Location:** Multiple nodes across network
- **Encryption:** Yes (AES-256-GCM)
- **Redundancy:** Yes (15 shards, 5 parity)
- **Access Speed:** Network-dependent
- **Use Case:** Files, media, backups

---

## How They Work Together

### Scenario 1: Sending a Text Message
```
1. User sends text message
   └─► Stored in pkg/storage (SQLite)
   └─► Relayed through network
   └─► Recipient stores in their pkg/storage
```

### Scenario 2: Sending an Image
```
1. User uploads image
   └─► Image encrypted (pkg/meshstorage)
   └─► Split into 15 shards (10 data + 5 parity)
   └─► Shards distributed to mesh nodes
   └─► Returns chunk ID and encryption key

2. Text message sent with chunk ID + key
   └─► Message stored in pkg/storage (SQLite)
   └─► Message includes chunk ID reference

3. Recipient downloads image
   └─► Fetches shards from mesh using chunk ID
   └─► Reconstructs and decrypts image
   └─► Message metadata in pkg/storage
```

### Scenario 3: Backup and Restore
```
1. User triggers backup
   └─► pkg/storage exports all data
   └─► Data encrypted by pkg/meshstorage
   └─► Stored across mesh network

2. User restores on new device
   └─► Downloads encrypted backup from mesh
   └─► Decrypts backup data
   └─► Imports into pkg/storage
```

---

## Storage Decision Matrix

| Content Type | Storage System | Reason |
|-------------|----------------|---------|
| Text messages | `pkg/storage` | Fast local access, frequent queries |
| Contact list | `pkg/storage` | Local data, privacy |
| Conversation metadata | `pkg/storage` | Fast queries, UI updates |
| Photos/Images | `pkg/meshstorage` | Large files, needs redundancy |
| Videos | `pkg/meshstorage` | Very large, decentralized CDN |
| Voice messages | `pkg/meshstorage` | Media content |
| Documents/PDFs | `pkg/meshstorage` | Large attachments |
| Profile avatars | `pkg/meshstorage` | Shared across users |
| Encrypted backups | `pkg/meshstorage` | Redundancy critical |
| Temporary cache | `pkg/storage` | Fast access |

---

## Database vs Mesh Storage

### When to Use `pkg/storage` (SQLite)

✅ **Use for:**
- Frequently accessed data
- Small data (< 1 MB)
- Data needed offline
- Chat messages and metadata
- Local user preferences
- Temporary cache

❌ **Don't use for:**
- Large files (> 1 MB)
- Media content
- Data needing redundancy
- Shared content across users

### When to Use `pkg/meshstorage` (P2P Mesh)

✅ **Use for:**
- Large files (> 1 MB)
- Media content (images, videos, audio)
- Content needing redundancy
- Shared files across users
- Long-term backups
- Decentralized distribution

❌ **Don't use for:**
- Frequently updated data
- Data needed for instant access
- Small metadata
- Temporary data

---

## Code Examples

### Example 1: Storing a Chat Message (pkg/storage)
```go
import "github.com/zentalk/protocol/pkg/storage"

// Initialize database
db, err := storage.NewDatabase("./messages.db")
if err != nil {
    return err
}

// Store message
msg := &storage.Message{
    From:      "user1_address",
    To:        "user2_address",
    Content:   "Hello!",
    Timestamp: time.Now().Unix(),
}

err = db.SaveMessage(msg)
```

### Example 2: Storing a File (pkg/meshstorage)
```go
import "github.com/zentalk/protocol/pkg/meshstorage"

// Create mesh storage node
config := &meshstorage.NodeConfig{
    Port:           9000,
    DataDir:        "./mesh-data",
    BootstrapPeers: []string{"/ip4/1.2.3.4/tcp/9000/p2p/..."},
}

node, err := meshstorage.NewDHTNode(ctx, config)
if err != nil {
    return err
}

storage := meshstorage.NewDistributedStorage(node, "./shards")

// Upload file
fileData := []byte("... large file content ...")
cid, err := storage.Store(ctx, fileData, userAddr, encryptionKey)
if err != nil {
    return err
}

// Store chunk ID reference in message database
msg := &storage.Message{
    From:        "user1_address",
    To:          "user2_address",
    Content:     "Check out this photo!",
    MediaChunkID: cid, // Link to mesh storage
    Timestamp:   time.Now().Unix(),
}
```

---

## Directory Structure

```
zentalk-protocol/
├── pkg/
│   ├── storage/              ← Local SQLite database
│   │   ├── database.go
│   │   ├── messages.go
│   │   ├── contacts.go
│   │   ├── conversations.go
│   │   └── relay_queue.go
│   │
│   └── meshstorage/          ← Distributed P2P storage
│       ├── node.go           (DHT node)
│       ├── distributed.go    (Coordination)
│       ├── storage.go        (Local shards)
│       ├── erasure.go        (Reed-Solomon)
│       ├── encryption.go     (AES-256-GCM)
│       ├── rpc.go            (P2P protocol)
│       ├── version.go        (Versioning)
│       ├── migration.go      (Migrations)
│       └── api/              (REST API)
│           ├── server.go
│           ├── upload.go
│           ├── download.go
│           ├── status.go
│           └── network.go
│
└── cmd/
    ├── relay/                ← Uses pkg/storage
    └── storage-node/         ← Uses pkg/meshstorage
        └── main.go
```

---

## Testing

Both storage systems have comprehensive test coverage:

### pkg/storage Tests
- Message CRUD operations
- Contact management
- Conversation tracking
- Relay queue operations

### pkg/meshstorage Tests (66 tests)
- Encryption/decryption
- Erasure coding
- Distributed storage
- RPC protocol
- Version migration
- Privacy guarantees
- Shard repair

Run tests:
```bash
# Test local storage
go test ./pkg/storage/...

# Test mesh storage
go test ./pkg/meshstorage/...

# Test all storage
go test ./pkg/storage/... ./pkg/meshstorage/...
```

---

## Performance Considerations

### pkg/storage (SQLite)
- **Read Latency:** < 1ms (local disk)
- **Write Latency:** < 5ms (local disk)
- **Concurrent Access:** Single writer, multiple readers
- **Scalability:** Limited by device storage

### pkg/meshstorage (P2P Mesh)
- **Upload Latency:** 100-500ms (network + encryption + distribution)
- **Download Latency:** 100-300ms (network + reconstruction + decryption)
- **Concurrent Access:** Unlimited (distributed)
- **Scalability:** Scales horizontally with more nodes

---

## Security Model

### pkg/storage
- **At-Rest:** Relies on device encryption (FileVault, BitLocker, etc.)
- **Access Control:** File system permissions
- **Threat Model:** Device compromise

### pkg/meshstorage
- **At-Rest:** AES-256-GCM encryption (always encrypted)
- **In-Transit:** Encrypted P2P connections
- **Access Control:** RSA signature verification
- **Threat Model:** Network adversary, malicious nodes
- **Zero-Knowledge:** Storage nodes cannot decrypt data

---

## Future Enhancements

### Planned Features
1. **Hybrid Caching** - Cache frequently accessed mesh content in pkg/storage
2. **Smart Sync** - Sync pkg/storage across user devices via mesh
3. **Incremental Backups** - Delta backups using both systems
4. **Content Deduplication** - Mesh-wide deduplication
5. **Cross-Network Bridges** - Bridge to other decentralized networks

---

## Conclusion

Both `pkg/storage` and `pkg/meshstorage` are **essential and complementary**:

- **`pkg/storage`** provides fast, local storage for chat data
- **`pkg/meshstorage`** provides secure, distributed storage for files

Together, they enable ZenTalk to offer:
✅ Fast messaging with local persistence
✅ Secure, redundant file storage
✅ No central servers
✅ Privacy-preserving architecture
✅ Resilience to node failures

---

**For questions or contributions, see the main README or open an issue.**
