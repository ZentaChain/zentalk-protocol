# How ZenTalk Mesh Network Works - Complete Guide

## Overview

The mesh network is a **decentralized storage system** where multiple storage nodes work together to store users' encrypted chat history. No single node has all the data, and the network can survive node failures.

---

## Network Architecture

```
                    ┌─────────────────────┐
                    │   User (Alice)      │
                    │   Web Browser       │
                    │   zentalk.com       │
                    └──────────┬──────────┘
                               │
                               │ HTTPS API calls
                               ↓
                    ┌─────────────────────┐
                    │  API Gateway        │
                    │  (ZenTalk Server)   │
                    │  - Encrypts E2EE    │
                    │  - Splits chunks    │
                    └──────────┬──────────┘
                               │
                               │ DHT Queries
                               ↓
        ┌──────────────────────┴──────────────────────┐
        │         MESH NETWORK (DHT)                  │
        │                                             │
        │  ┌─────────┐    ┌─────────┐    ┌─────────┐│
        │  │ Node A  │────│ Node B  │────│ Node C  ││
        │  │ (8001)  │    │ (8002)  │    │ (8003)  ││
        │  └────┬────┘    └────┬────┘    └────┬────┘│
        │       │              │              │     │
        │  ┌────┴────┐    ┌────┴────┐    ┌────┴────┐│
        │  │ Node D  │────│ Node E  │────│ Node F  ││
        │  │ (8004)  │    │ (8005)  │    │ (8006)  ││
        │  └────┬────┘    └────┬────┘    └────┬────┘│
        │       │              │              │     │
        │       └──────────────┴──────────────┘     │
        │                                           │
        └───────────────────────────────────────────┘
```

---

## Step-by-Step: How It Works

### 1. Storage Node Joins Network

**Scenario**: Operator starts a new storage node

```bash
# Operator runs storage node
./storage-node --port 8007 --bootstrap 8.8.8.8:8001

# What happens:
```

```
Step 1: Node generates unique ID
┌──────────────────────────────────────┐
│ Storage Node 8007                    │
│ Generate ID = SHA256(public_key)     │
│ Node ID: 0x4f2a9b1c...               │
└──────────────────────────────────────┘

Step 2: Connect to bootstrap node
┌──────────────────────────────────────┐
│ Node 8007 → Bootstrap Node (8001)    │
│ "Hello, I'm 0x4f2a9b1c..."           │
│ "What nodes do you know?"            │
└──────────────────────────────────────┘

Step 3: Bootstrap responds with known nodes
┌──────────────────────────────────────┐
│ Bootstrap Node 8001 → Node 8007      │
│ "Here are nodes close to you:"       │
│ - Node 8002 (0x3a1f8e...)            │
│ - Node 8003 (0x5c2d9a...)            │
│ - Node 8004 (0x6b3e8f...)            │
└──────────────────────────────────────┘

Step 4: Node 8007 pings each node
┌──────────────────────────────────────┐
│ Node 8007 → Node 8002                │
│ PING "Are you alive?"                │
│ PONG "Yes, I'm alive"                │
│                                      │
│ → Adds Node 8002 to routing table   │
└──────────────────────────────────────┘

Step 5: Node is now part of the mesh
┌──────────────────────────────────────┐
│ Node 8007 Routing Table:             │
│ Bucket 0: [Node 8002, Node 8003]     │
│ Bucket 1: [Node 8004]                │
│ ...                                  │
│ Status: ONLINE, ready to store data  │
└──────────────────────────────────────┘
```

---

### 2. User Stores Chat Message

**Scenario**: Alice sends "Hello Bob" message

```
Step 1: User sends message via web app
┌──────────────────────────────────────┐
│ Alice's Browser (zentalk.com)        │
│ Message: "Hello Bob"                 │
│ ↓ Client-side E2EE encryption        │
│ Encrypted: 0x8a3f9b2c... (512 bytes) │
└──────────────────────────────────────┘
         │
         │ POST /api/send
         ↓
┌──────────────────────────────────────┐
│ API Gateway                          │
│ Receives encrypted message           │
│ User: alice (0xabc123...)            │
└──────────────────────────────────────┘

Step 2: Gateway splits into chunks (Erasure Coding)
┌──────────────────────────────────────┐
│ Erasure Coding (10 data + 5 parity)  │
│                                      │
│ Input: 512 bytes encrypted data      │
│                                      │
│ Split into 10 chunks (51.2 bytes):   │
│  Chunk 0: 0x8a3f9b... (data)        │
│  Chunk 1: 0x2c1d8e... (data)        │
│  Chunk 2: 0x9f4a2b... (data)        │
│  ...                                 │
│  Chunk 9: 0x5e7c3a... (data)        │
│                                      │
│ Generate 5 parity chunks:            │
│  Chunk 10: 0x3d2f8a... (parity)     │
│  Chunk 11: 0x7b5c1e... (parity)     │
│  Chunk 12: 0x4a9d2f... (parity)     │
│  Chunk 13: 0x8e3b5c... (parity)     │
│  Chunk 14: 0x2f9a4d... (parity)     │
└──────────────────────────────────────┘

Step 3: Find storage nodes using DHT
┌──────────────────────────────────────┐
│ DHT Query                            │
│                                      │
│ Key = SHA256("alice" + 0xabc123...)  │
│     = 0xd4f6a8b2c1e3...              │
│                                      │
│ Query DHT: "Find 15 closest nodes"   │
└──────────────────────────────────────┘
         │
         ↓
┌──────────────────────────────────────┐
│ DHT Returns 15 Storage Nodes:       │
│                                      │
│ Distance to 0xd4f6a8b2c1e3...        │
│  1. Node 8001 (dist: 0x0012)         │
│  2. Node 8002 (dist: 0x0034)         │
│  3. Node 8003 (dist: 0x0056)         │
│  4. Node 8004 (dist: 0x0078)         │
│  ...                                 │
│ 15. Node 8015 (dist: 0x0234)         │
└──────────────────────────────────────┘

Step 4: Distribute chunks to nodes
┌──────────────────────────────────────┐
│ API Gateway → Storage Nodes          │
│                                      │
│ Node 8001 ← Chunk 0 (data)           │
│ Node 8002 ← Chunk 1 (data)           │
│ Node 8003 ← Chunk 2 (data)           │
│ Node 8004 ← Chunk 3 (data)           │
│ Node 8005 ← Chunk 4 (data)           │
│ Node 8006 ← Chunk 5 (data)           │
│ Node 8007 ← Chunk 6 (data)           │
│ Node 8008 ← Chunk 7 (data)           │
│ Node 8009 ← Chunk 8 (data)           │
│ Node 8010 ← Chunk 9 (data)           │
│ Node 8011 ← Chunk 10 (parity)        │
│ Node 8012 ← Chunk 11 (parity)        │
│ Node 8013 ← Chunk 12 (parity)        │
│ Node 8014 ← Chunk 13 (parity)        │
│ Node 8015 ← Chunk 14 (parity)        │
└──────────────────────────────────────┘

Step 5: Each node stores its chunk
┌──────────────────────────────────────┐
│ Node 8001 Storage (SQLite)           │
│                                      │
│ INSERT INTO chunks VALUES (          │
│   user_addr: 'alice_0xabc123...',    │
│   chunk_id: 0,                       │
│   data: 0x8a3f9b...,                 │
│   stored_at: 1705320000,             │
│   size: 51                           │
│ )                                    │
│                                      │
│ ✓ Chunk stored successfully          │
└──────────────────────────────────────┘

Step 6: Nodes send acknowledgment
┌──────────────────────────────────────┐
│ All 15 nodes → API Gateway           │
│                                      │
│ Node 8001: "Chunk 0 stored ✓"        │
│ Node 8002: "Chunk 1 stored ✓"        │
│ ...                                  │
│ Node 8015: "Chunk 14 stored ✓"       │
│                                      │
│ API Gateway: "All chunks stored!"    │
└──────────────────────────────────────┘
```

---

### 3. User Retrieves Chat History

**Scenario**: Alice opens her chat

```
Step 1: Alice requests chat history
┌──────────────────────────────────────┐
│ Alice's Browser                      │
│ GET /api/chats                       │
│ X-Wallet-Address: 0xabc123...        │
└──────────────────────────────────────┘
         │
         ↓
┌──────────────────────────────────────┐
│ API Gateway                          │
│ "Fetch Alice's chat history"         │
└──────────────────────────────────────┘

Step 2: Find storage nodes (same as before)
┌──────────────────────────────────────┐
│ DHT Query                            │
│ Key = SHA256("alice" + 0xabc123...)  │
│     = 0xd4f6a8b2c1e3...              │
│                                      │
│ Returns same 15 nodes                │
└──────────────────────────────────────┘

Step 3: Request chunks from nodes
┌──────────────────────────────────────┐
│ API Gateway → Storage Nodes          │
│                                      │
│ "Send me chunks for alice"           │
│                                      │
│ Request to all 15 nodes in parallel  │
└──────────────────────────────────────┘

Step 4: Nodes respond with chunks
┌──────────────────────────────────────┐
│ Responses:                           │
│                                      │
│ Node 8001 → Chunk 0 ✓                │
│ Node 8002 → Chunk 1 ✓                │
│ Node 8003 → Chunk 2 ✓                │
│ Node 8004 → Chunk 3 ✓                │
│ Node 8005 → TIMEOUT ✗ (offline)      │
│ Node 8006 → Chunk 5 ✓                │
│ Node 8007 → Chunk 6 ✓                │
│ Node 8008 → TIMEOUT ✗ (offline)      │
│ Node 8009 → Chunk 8 ✓                │
│ Node 8010 → Chunk 9 ✓                │
│ Node 8011 → Chunk 10 ✓ (parity)      │
│ Node 8012 → Chunk 11 ✓ (parity)      │
│ Node 8013 → TIMEOUT ✗ (offline)      │
│ Node 8014 → Chunk 13 ✓ (parity)      │
│ Node 8015 → Chunk 14 ✓ (parity)      │
│                                      │
│ Received: 12 chunks (need 10)        │
│ Missing: Chunks 4, 7, 12             │
└──────────────────────────────────────┘

Step 5: Reconstruct data (Erasure Coding)
┌──────────────────────────────────────┐
│ Erasure Coding Reconstruction        │
│                                      │
│ Have 12 chunks (need only 10)        │
│  - 8 data chunks (0,1,2,3,6,8,9)     │
│  - 4 parity chunks (10,11,13,14)     │
│                                      │
│ Reconstruct missing chunks 4 and 7:  │
│  Chunk 4 = f(10,11,13,14)            │
│  Chunk 7 = f(10,11,13,14)            │
│                                      │
│ Now have all 10 data chunks!         │
│                                      │
│ Combine: Encrypted message restored  │
│ Output: 0x8a3f9b2c... (512 bytes)    │
└──────────────────────────────────────┘

Step 6: Send to user, decrypt client-side
┌──────────────────────────────────────┐
│ API Gateway → Alice's Browser        │
│ Encrypted: 0x8a3f9b2c...             │
└──────────────────────────────────────┘
         │
         ↓
┌──────────────────────────────────────┐
│ Alice's Browser                      │
│ Client-side decryption (E2EE)        │
│ Plaintext: "Hello Bob"               │
│                                      │
│ ✓ Message displayed!                 │
└──────────────────────────────────────┘
```

**Key Point**: Even though 3 nodes were offline (8005, 8008, 8013), we still reconstructed the full message!

---

### 4. Node Leaves Network (Fault Tolerance)

**Scenario**: Node 8005 goes permanently offline

```
Step 1: Network detects node failure
┌──────────────────────────────────────┐
│ Periodic Health Check (every 5 min)  │
│                                      │
│ Node 8001 PING → Node 8005           │
│ TIMEOUT (no response)                │
│                                      │
│ Retry 1: TIMEOUT                     │
│ Retry 2: TIMEOUT                     │
│ Retry 3: TIMEOUT                     │
│                                      │
│ Mark Node 8005 as DEAD               │
└──────────────────────────────────────┘

Step 2: Find affected users
┌──────────────────────────────────────┐
│ Query: "Which users had data on 8005?"│
│                                      │
│ DHT reverse lookup:                  │
│  - Alice (Chunk 4)                   │
│  - Bob (Chunk 2)                     │
│  - Carol (Chunk 9)                   │
│  ...                                 │
└──────────────────────────────────────┘

Step 3: Re-replicate missing chunks
┌──────────────────────────────────────┐
│ For Alice:                           │
│                                      │
│ 1. Fetch remaining chunks from       │
│    other 14 nodes                    │
│                                      │
│ 2. Reconstruct Chunk 4 using         │
│    erasure coding                    │
│                                      │
│ 3. Find new storage node for         │
│    Chunk 4 (DHT finds Node 8016)     │
│                                      │
│ 4. Store reconstructed Chunk 4       │
│    on Node 8016                      │
│                                      │
│ ✓ Redundancy restored!               │
└──────────────────────────────────────┘

Step 4: Update routing tables
┌──────────────────────────────────────┐
│ All nodes remove Node 8005 from      │
│ their routing tables                 │
│                                      │
│ Node 8016 takes over storage for     │
│ data previously on Node 8005         │
└──────────────────────────────────────┘
```

---

### 5. DHT Routing (How Nodes Find Each Other)

**Scenario**: Node 8001 needs to find Node 8015

```
Kademlia XOR Distance:

Node 8001 ID: 0x4f2a9b1c...
Node 8015 ID: 0x3a1f8e2d...

XOR Distance:
  0x4f2a9b1c...
⊕ 0x3a1f8e2d...
─────────────────
  0x75351531...  ← Distance value

Closer nodes have smaller XOR distance.

Routing Table (Node 8001):
┌──────────────────────────────────────┐
│ Bucket 0 (distance 2^0 - 2^1):       │
│   - Node 8002 (dist: 0x0012)         │
│                                      │
│ Bucket 1 (distance 2^1 - 2^2):       │
│   - Node 8003 (dist: 0x0034)         │
│   - Node 8004 (dist: 0x0056)         │
│                                      │
│ Bucket 2 (distance 2^2 - 2^3):       │
│   - Node 8005 (dist: 0x0078)         │
│   ...                                │
│                                      │
│ Bucket 255 (farthest nodes):         │
│   - Node 8015 (dist: 0x75351531...)  │
└──────────────────────────────────────┘

Finding Node 8015:
Step 1: Check routing table, not found locally
Step 2: Ask closest known nodes
  Node 8001 → Node 8002: "Know anyone close to 0x3a1f8e2d?"
  Node 8002 → "Yes, Node 8007"
Step 3: Ask Node 8007
  Node 8001 → Node 8007: "Know anyone close to 0x3a1f8e2d?"
  Node 8007 → "Yes, Node 8015!"
Step 4: Connect to Node 8015
  Node 8001 → Node 8015: "Found you!"
```

---

## Complete Message Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Alice sends message "Hello Bob"                          │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Browser encrypts (E2EE): 0x8a3f9b... (512 bytes)         │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. API Gateway splits into 15 chunks (10 data + 5 parity)   │
│    Chunk 0: 51 bytes                                        │
│    Chunk 1: 51 bytes                                        │
│    ...                                                      │
│    Chunk 14: 51 bytes                                       │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. DHT finds 15 storage nodes closest to Alice's address    │
│    Hash(alice) = 0xd4f6a8...                                │
│    Nodes: 8001, 8002, ..., 8015                             │
└───────────────────────┬─────────────────────────────────────┘
                        │
            ┌───────────┴───────────┐
            │                       │
            ↓                       ↓
    ┌──────────────┐        ┌──────────────┐
    │ Node 8001    │        │ Node 8002    │  ...→ Node 8015
    │ Store        │        │ Store        │
    │ Chunk 0      │        │ Chunk 1      │
    └──────┬───────┘        └──────┬───────┘
           │                       │
           └───────────┬───────────┘
                       │
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. All nodes acknowledge storage                            │
│    15/15 chunks stored successfully                         │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. API Gateway confirms to Alice                            │
│    "Message sent ✓"                                         │
└─────────────────────────────────────────────────────────────┘

Later: Alice retrieves message
┌─────────────────────────────────────────────────────────────┐
│ 7. Alice requests chat history                              │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 8. API Gateway queries DHT for Alice's storage nodes        │
└───────────────────────┬─────────────────────────────────────┘
                        │
            ┌───────────┴───────────┐
            │                       │
            ↓                       ↓
    ┌──────────────┐        ┌──────────────┐
    │ Node 8001    │        │ Node 8002    │  ...→ Node 8015
    │ Return       │        │ Return       │
    │ Chunk 0 ✓    │        │ Chunk 1 ✓    │
    └──────┬───────┘        └──────┬───────┘
           │                       │
           └───────────┬───────────┘
                       │
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ 9. Received 12/15 chunks (3 nodes offline)                  │
│    Enough to reconstruct! (need only 10)                    │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 10. Erasure coding reconstructs full encrypted message      │
│     Output: 0x8a3f9b... (512 bytes)                         │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 11. Send to Alice's browser                                 │
│     Browser decrypts client-side                            │
│     Display: "Hello Bob"                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Mesh Network Properties

### 1. **Decentralization**
- No single point of failure
- No central server controls data
- Nodes join/leave freely

### 2. **Fault Tolerance**
```
Can lose 5 out of 15 nodes:
✓ Still have 10 chunks (enough to reconstruct)
✓ Automatic re-replication to new nodes
✓ No data loss
```

### 3. **Self-Healing**
```
Node goes offline:
→ Network detects failure (health checks)
→ Reconstructs missing chunks
→ Stores on new node
→ Redundancy restored
```

### 4. **Scalability**
```
Network size: 100+ nodes
Each user's data: Stored on 15 random nodes
Load balancing: Automatic (DHT assigns nodes)
```

### 5. **Privacy**
```
Each node stores:
- Encrypted chunk (unreadable)
- User address (visible, for routing)
- Chunk ID (visible)

Nodes CANNOT:
- Decrypt message content
- Know full message structure
- Identify sender
```

---

## Network Topology Example

### Small Network (10 nodes):

```
         Node A
        /      \
    Node B    Node C
      |    ×    |
    Node D    Node E
      |    ×    |
    Node F    Node G
       \      /
        Node H
        /    \
    Node I   Node J

× = Cross-connections (each node knows 3-5 neighbors)
```

### Large Network (100 nodes):

```
Each node maintains ~log2(N) connections
For 100 nodes: ~7 connections per node

Finding any node: O(log N) hops
For 100 nodes: ~7 hops maximum

Example path:
Node 1 → Node 23 → Node 67 → Node 89 → Node 100
(5 hops to reach any node)
```

---

## Comparison: Centralized vs Mesh

### Centralized (Traditional Server):

```
All users → Single Server
               ↓
         Single Database

Problems:
✗ Single point of failure (server down = service down)
✗ Server operator can read all messages
✗ Server operator can delete messages
✗ High costs for server operator
```

### Mesh Network (ZenTalk):

```
User 1 → Nodes [1,5,9,12,...]
User 2 → Nodes [3,7,11,15,...]
User 3 → Nodes [2,6,10,14,...]

Advantages:
✓ Distributed (5 nodes can fail, data still recoverable)
✓ Nodes cannot read messages (E2EE encrypted)
✓ Nodes get rewarded (blockchain incentives)
✓ No single operator controls network
```

---

## Practical Example: Alice's Chat History

```
Alice has 100 messages (50 KB total)

Storage in Mesh:
├─ Encrypted: 50 KB (client-side E2EE)
├─ Erasure coded: 75 KB (10+5 chunks, 1.5x overhead)
├─ Distributed across: 15 nodes
│
├─ Node 8001 stores: 5 KB (Chunk 0)
├─ Node 8002 stores: 5 KB (Chunk 1)
├─ ...
└─ Node 8015 stores: 5 KB (Chunk 14)

Retrieval:
├─ Query 15 nodes in parallel
├─ Need 10 responses to reconstruct
├─ Latency: ~200ms (parallel requests)
└─ Result: Full 50 KB chat history
```

---

## Summary: How Mesh Network Works

1. **Nodes join** by connecting to bootstrap node and building routing table
2. **Data stored** by splitting into chunks and distributing to DHT-selected nodes
3. **Data retrieved** by querying nodes and reconstructing from available chunks
4. **Failures handled** by re-replicating to new nodes automatically
5. **Routing** uses Kademlia XOR distance to find closest nodes
6. **Privacy** maintained through E2EE encryption (nodes store encrypted chunks)

**Result**: Decentralized, fault-tolerant, privacy-preserving storage network where nodes earn rewards for participation.

---

## Next: Implementation

See `MESH_STORAGE_IMPLEMENTATION.md` for code examples and how to build this system.
