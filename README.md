# ZenTalk Protocol

> Decentralized communication standards, routing mechanisms, and specifications for the ZenTalk network

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## ⚠️ UNDER DEVELOPMENT - NOT FOR PRODUCTION

**This protocol is currently under active development and is NOT ready for production use.**

- ❌ Protocol specifications may change
- ❌ Security audits not completed
- ❌ Not finalized for production networks

---

## Overview

The ZenTalk Protocol defines the standards and mechanisms for a decentralized, privacy-first messaging network. This repository contains the complete protocol specifications, architecture documentation, and technical details for developers who want to understand or implement the protocol.

## Key Components

The protocol architecture includes:

1. **Mesh Router** — Determines optimal delivery paths across the network
2. **ZKP Routing** — Verifies routes while protecting message content (zero-knowledge proofs)
3. **Onion Relayer** — Applies layered encryption for user anonymity
4. **Pontion Layer** — Manages node coordination and network topology
5. **DHT Storage** — Distributes messages across nodes with redundancy
6. **CHAIN Rewards** — Incentivizes network participation with token rewards

## Documentation

### Core Architecture

- **[ARCHITECTURE.md](docs/ARCHITECTURE.md)** - Complete system architecture overview
- **[MULTI_RELAY_ARCHITECTURE.md](docs/MULTI_RELAY_ARCHITECTURE.md)** - Multi-hop relay routing design
- **[STORAGE_ARCHITECTURE.md](docs/STORAGE_ARCHITECTURE.md)** - Distributed storage architecture

### Networking & Privacy

- **[HOW_MESHNET_WORKS.md](docs/HOW_MESHNET_WORKS.md)** - Mesh network topology and routing
- **[PRIVACY_FAQ.md](docs/PRIVACY_FAQ.md)** - Privacy features and guarantees
- **[RELAY_FAQ.md](docs/RELAY_FAQ.md)** - Relay server operations and privacy
- **[IPFS_REFERENCES_EXPLAINED.md](docs/IPFS_REFERENCES_EXPLAINED.md)** - IPFS integration details

### Security

- **[SECURITY_ANALYSIS.md](docs/SECURITY_ANALYSIS.md)** - Comprehensive security analysis
  - Encryption mechanisms (Double Ratchet, X3DH)
  - Threat models and attack vectors
  - Privacy guarantees

### Technical Specifications

- **[PHASE3_TECHNICAL_SPEC.md](docs/PHASE3_TECHNICAL_SPEC.md)** - Complete Phase 3 technical specifications
- **[PHASE3_CHAIN_TOKEN_INTEGRATION.md](docs/PHASE3_CHAIN_TOKEN_INTEGRATION.md)** - CHAIN token integration and rewards

## Core Features

### End-to-End Encryption

- **Double Ratchet Algorithm** - Perfect forward secrecy with automatic key rotation
- **X3DH Key Agreement** - Secure initial key exchange without prior communication
- **Onion Routing** - Multi-layer encryption hiding sender and receiver

### Decentralized Architecture

- **No Central Servers** - Fully distributed peer-to-peer network
- **DHT-Based Discovery** - Decentralized peer and relay discovery
- **Multi-Relay Routing** - Messages route through multiple relays for privacy

### Privacy Protection

- **Zero Knowledge** - Relays cannot decrypt message content
- **Metadata Protection** - Onion routing hides sender/receiver relationship
- **Perfect Forward Secrecy** - Past messages remain secure even if keys compromised

### Network Incentives

- **CHAIN Token Rewards** - Operators earn rewards for:
  - Running relay nodes
  - Providing mesh storage
  - Maintaining high uptime
  - Supporting network operations

## Protocol Layers

```
┌─────────────────────────────────────────────────────┐
│  Application Layer (Client Apps)                   │
├─────────────────────────────────────────────────────┤
│  Message Protocol (Double Ratchet + X3DH)          │
├─────────────────────────────────────────────────────┤
│  Routing Layer (Onion Routing)                     │
├─────────────────────────────────────────────────────┤
│  Network Layer (DHT + Mesh Topology)               │
├─────────────────────────────────────────────────────┤
│  Transport Layer (TCP/TLS)                         │
└─────────────────────────────────────────────────────┘
```

## Message Flow

```
Alice                Relay 1              Relay 2              Relay 3              Bob
  │                    │                    │                    │                    │
  ├─ Encrypt(3x) ─────>│                    │                    │                    │
  │                    ├─ Decrypt(1) ──────>│                    │                    │
  │                    │                    ├─ Decrypt(1) ──────>│                    │
  │                    │                    │                    ├─ Decrypt(1) ──────>│
  │                    │                    │                    │                    ├─ Read
```

## Implementations

### Reference Implementations

- **[ZenTalk API](https://github.com/ZentaChain/zentalk-api)** - Client API server implementation (Go)
- **[ZenTalk Node](https://github.com/ZentaChain/zentalk-node)** - Node operator software (Go)

### Integration

To integrate with the ZenTalk network:

1. **Build a Client**: Use the ZenTalk API server
2. **Run a Node**: Deploy ZenTalk Node to earn rewards
3. **Implement Protocol**: Follow specifications in `/docs`

## Contributing

We welcome contributions to the protocol specifications! Please:

1. Read the existing documentation
2. Propose changes via GitHub Issues
3. Submit pull requests with clear explanations

## Roadmap

- [ ] Finalize protocol specifications v1.0
- [ ] Complete security audit
- [ ] Add formal verification proofs
- [ ] Multi-language protocol documentation
- [ ] Reference implementations in multiple languages

## License

MIT License - see LICENSE file for details

## Community

- **Website**: [zentachain.io](https://zentachain.io)
- **Discord**: [discord.gg/zentachain](https://discord.gg/zentachain)
- **Telegram**: [t.me/ZentaChain](https://t.me/ZentaChain)
- **Email**: info@zentachain.io

## Links

- [ZenTalk API Repository](https://github.com/ZentaChain/zentalk-api)
- [ZenTalk Node Repository](https://github.com/ZentaChain/zentalk-node)
- [Block Explorer](https://explorer.zentachain.io)
- [Documentation](https://docs.zentalk.chat)

---

**A secure, private, and resilient messaging protocol for the decentralized web** 🔐
