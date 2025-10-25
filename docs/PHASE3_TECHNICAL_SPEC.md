# Phase 3: Blockchain Integration - Technical Specification

**Version**: 1.0
**Status**: Draft
**Last Updated**: 2025-01-20
**Authors**: ZenTalk Development Team

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Architecture](#system-architecture)
3. [Smart Contract Specifications](#smart-contract-specifications)
4. [Merkle Proof System](#merkle-proof-system)
5. [Challenge-Response Protocol](#challenge-response-protocol)
6. [Token Economics](#token-economics)
7. [API Specifications](#api-specifications)
8. [Integration with Phase 2](#integration-with-phase-2)
9. [Security Considerations](#security-considerations)
10. [Testing Strategy](#testing-strategy)
11. [Deployment Plan](#deployment-plan)
12. [Performance Requirements](#performance-requirements)

---

## Executive Summary

### Objectives

Phase 3 adds blockchain-based economic incentives to the ZenTalk mesh storage network, enabling:

1. **Storage Node Rewards**: Nodes earn tokens for providing storage
2. **Proof-of-Storage**: Cryptographic verification that data is actually stored
3. **Economic Security**: Staking and slashing mechanisms prevent attacks
4. **Payment System**: Users pay for storage, nodes earn from providing it
5. **Decentralized Governance**: Token holders vote on network parameters

### Success Criteria

- ✅ Storage nodes can stake tokens and earn rewards
- ✅ Nodes can prove they're storing data without revealing it
- ✅ Random challenges verify ongoing storage availability
- ✅ Failed challenges trigger automatic slashing and re-replication
- ✅ Gas costs < $0.01 per challenge verification
- ✅ Challenge response time < 5 minutes
- ✅ 99.9% challenge success rate in normal operation

### Key Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Minimum Stake | 1,000 tokens | Per storage node |
| Challenge Frequency | Every 24-48 hours | Per shard |
| Challenge Response Time | < 5 minutes | 95th percentile |
| Slashing Penalty | 10% of stake | Per failed challenge |
| Daily Reward Pool | 1,000 tokens | Adjustable by governance |
| Proof Size | < 1 KB | Merkle proof overhead |
| Gas Cost per Challenge | < 50,000 gas | On Ethereum L2 |

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     ZenTalk Users                           │
│                  (Web Browser / Mobile)                      │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ HTTPS API
                        ↓
┌─────────────────────────────────────────────────────────────┐
│                   API Gateway Server                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  E2EE Layer  │→ │ Erasure Code │→ │   Merkle     │      │
│  │  (Phase 0)   │  │   (Phase 2)  │  │  Tree Gen    │      │
│  └──────────────┘  └──────────────┘  └──────┬───────┘      │
│                                              │               │
│  ┌──────────────────────────────────────────┼───────────┐   │
│  │        Blockchain Client                 │           │   │
│  │  - Contract interactions                 │           │   │
│  │  - Merkle root submission                │           │   │
│  │  - Payment handling                      ↓           │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ libp2p + Blockchain Events
                        ↓
┌─────────────────────────────────────────────────────────────┐
│              Mesh Storage Network (DHT)                      │
│                                                              │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│  │  Node 1   │  │  Node 2   │  │  Node 3   │  ... × 100    │
│  │ ┌───────┐ │  │ ┌───────┐ │  │ ┌───────┐ │               │
│  │ │Storage│ │  │ │Storage│ │  │ │Storage│ │               │
│  │ │SQLite │ │  │ │SQLite │ │  │ │SQLite │ │               │
│  │ └───────┘ │  │ └───────┘ │  │ └───────┘ │               │
│  │ ┌───────┐ │  │ ┌───────┐ │  │ ┌───────┐ │               │
│  │ │ Chain │ │  │ │ Chain │ │  │ │ Chain │ │               │
│  │ │Client │ │  │ │Client │ │  │ │Client │ │               │
│  │ └───────┘ │  │ └───────┘ │  │ └───────┘ │               │
│  │ ┌───────┐ │  │ ┌───────┐ │  │ ┌───────┐ │               │
│  │ │Merkle │ │  │ │Merkle │ │  │ │Merkle │ │               │
│  │ │Prover │ │  │ │Prover │ │  │ │Prover │ │               │
│  │ └───────┘ │  │ └───────┘ │  │ └───────┘ │               │
│  └───────────┘  └───────────┘  └───────────┘               │
│         │              │              │                      │
└─────────┼──────────────┼──────────────┼──────────────────────┘
          │              │              │
          │ On-Chain Events & Transactions
          ↓              ↓              ↓
┌─────────────────────────────────────────────────────────────┐
│               Blockchain Layer (Ethereum L2)                 │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           Smart Contracts                            │   │
│  │                                                      │   │
│  │  ┌────────────────┐  ┌────────────────┐            │   │
│  │  │ StorageRewards │  │ ProofVerifier  │            │   │
│  │  │                │  │                │            │   │
│  │  │ - Node staking │  │ - Merkle proof │            │   │
│  │  │ - Reward calc  │  │   verification │            │   │
│  │  │ - Slashing     │  │ - Challenge    │            │   │
│  │  └────────────────┘  │   validation   │            │   │
│  │                      └────────────────┘            │   │
│  │                                                      │   │
│  │  ┌────────────────┐  ┌────────────────┐            │   │
│  │  │TokenDistributor│  │   Governance   │            │   │
│  │  │                │  │                │            │   │
│  │  │ - Daily rewards│  │ - Parameter    │            │   │
│  │  │ - Fee handling │  │   updates      │            │   │
│  │  └────────────────┘  │ - DAO voting   │            │   │
│  │                      └────────────────┘            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Token Contract (ERC-20)                 │   │
│  │  - ZENT token                                        │   │
│  │  - Transfer, stake, burn functions                  │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Component Interactions

```
Storage Flow with Blockchain:

1. User uploads data
   └─> API Gateway encrypts (E2EE)
       └─> Erasure encode (10+5 shards)
           └─> Generate Merkle tree
               └─> Submit root hash to blockchain
                   └─> Create storage contract
                       └─> Distribute shards to nodes
                           └─> Nodes store locally + merkle proofs

2. Random challenge (every 24h)
   └─> Smart contract emits ChallengeEvent
       └─> Node listens for event
           └─> Node retrieves shard from SQLite
               └─> Node generates Merkle proof
                   └─> Node submits proof to contract
                       └─> Contract verifies proof
                           ├─> Valid: Increase reputation, earn bonus
                           └─> Invalid: Slash stake, trigger re-replication

3. Daily reward distribution
   └─> Smart contract calculates rewards
       └─> Based on: storage provided, uptime, challenges passed
           └─> Distribute tokens to nodes
               └─> Nodes can unstake after lockup period
```

---

## Smart Contract Specifications

### 1. StorageRewards Contract

**Purpose**: Main contract for node registration, staking, and reward distribution

#### State Variables

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract StorageRewards {
    // --- Constants ---
    uint256 public constant MINIMUM_STAKE = 1000 * 10**18; // 1,000 tokens
    uint256 public constant UNSTAKE_LOCKUP = 90 days;
    uint256 public constant CHALLENGE_TIMEOUT = 5 minutes;
    uint256 public constant SLASH_PERCENTAGE = 10; // 10% of stake

    // --- State Variables ---

    // Node registry
    struct Node {
        address owner;              // Node operator address
        bytes32 nodeID;             // libp2p peer ID hash
        uint256 stakedAmount;       // Amount of tokens staked
        uint256 storageCapacityGB;  // Total storage capacity
        uint256 storageUsedGB;      // Currently used storage
        uint256 successfulChallenges;
        uint256 failedChallenges;
        uint256 totalEarnedRewards;
        uint256 registeredAt;       // Block timestamp
        uint256 unstakeRequestedAt; // When unstake was requested
        bool isActive;              // Currently accepting storage
        uint256 reputationScore;    // 0-10000 (100.00%)
    }

    mapping(address => Node) public nodes;
    mapping(bytes32 => address) public nodeIDToAddress;

    // Storage contracts
    struct StorageContract {
        bytes32 contractID;         // Unique contract ID
        address user;               // User who owns the data
        bytes32 merkleRoot;         // Root hash of data Merkle tree
        uint256 dataSizeBytes;      // Total data size
        uint256 shardCount;         // Number of shards (15)
        uint256 durationBlocks;     // How long to store
        uint256 rewardPoolTokens;   // Total tokens for this contract
        uint256 createdBlock;       // When contract was created
        uint256 expiresBlock;       // When contract expires
        bytes32[] assignedNodeIDs;  // Which nodes store shards
        bool isActive;              // Is contract still valid
    }

    mapping(bytes32 => StorageContract) public storageContracts;
    bytes32[] public activeContracts;

    // Rewards tracking
    uint256 public dailyRewardPool;          // Tokens distributed daily
    uint256 public lastRewardDistribution;   // Last distribution timestamp
    uint256 public totalNetworkStorageGB;    // Sum of all node storage

    // Token reference
    IERC20 public rewardToken;

    // --- Events ---

    event NodeRegistered(
        address indexed owner,
        bytes32 indexed nodeID,
        uint256 stakedAmount,
        uint256 storageCapacityGB
    );

    event NodeUnstakeRequested(
        address indexed owner,
        bytes32 indexed nodeID,
        uint256 unstakeDate
    );

    event NodeUnstaked(
        address indexed owner,
        bytes32 indexed nodeID,
        uint256 returnedAmount
    );

    event StorageContractCreated(
        bytes32 indexed contractID,
        address indexed user,
        bytes32 merkleRoot,
        uint256 dataSizeBytes,
        uint256 rewardPoolTokens
    );

    event ChallengeIssued(
        bytes32 indexed contractID,
        bytes32 indexed nodeID,
        uint256 shardIndex,
        uint256 deadline
    );

    event ChallengeResolved(
        bytes32 indexed contractID,
        bytes32 indexed nodeID,
        bool success,
        uint256 timestamp
    );

    event NodeSlashed(
        address indexed owner,
        bytes32 indexed nodeID,
        uint256 slashedAmount,
        string reason
    );

    event RewardsDistributed(
        uint256 totalAmount,
        uint256 nodeCount,
        uint256 timestamp
    );

    // --- Constructor ---

    constructor(address _rewardToken, uint256 _dailyRewardPool) {
        rewardToken = IERC20(_rewardToken);
        dailyRewardPool = _dailyRewardPool;
        lastRewardDistribution = block.timestamp;
    }
}
```

#### Core Functions

```solidity
// --- Node Management Functions ---

/**
 * @notice Register a new storage node with stake
 * @param nodeID libp2p peer ID (hashed)
 * @param storageCapacityGB Total storage capacity in GB
 */
function registerNode(
    bytes32 nodeID,
    uint256 storageCapacityGB
) external payable {
    require(msg.value >= MINIMUM_STAKE, "Insufficient stake");
    require(nodes[msg.sender].owner == address(0), "Already registered");
    require(nodeIDToAddress[nodeID] == address(0), "NodeID already used");

    nodes[msg.sender] = Node({
        owner: msg.sender,
        nodeID: nodeID,
        stakedAmount: msg.value,
        storageCapacityGB: storageCapacityGB,
        storageUsedGB: 0,
        successfulChallenges: 0,
        failedChallenges: 0,
        totalEarnedRewards: 0,
        registeredAt: block.timestamp,
        unstakeRequestedAt: 0,
        isActive: true,
        reputationScore: 10000 // Start with perfect reputation
    });

    nodeIDToAddress[nodeID] = msg.sender;
    totalNetworkStorageGB += storageCapacityGB;

    emit NodeRegistered(msg.sender, nodeID, msg.value, storageCapacityGB);
}

/**
 * @notice Request to unstake and leave the network
 */
function requestUnstake() external {
    Node storage node = nodes[msg.sender];
    require(node.owner != address(0), "Not registered");
    require(node.storageUsedGB == 0, "Still storing data");
    require(node.unstakeRequestedAt == 0, "Already requested");

    node.unstakeRequestedAt = block.timestamp;
    node.isActive = false;

    emit NodeUnstakeRequested(
        msg.sender,
        node.nodeID,
        block.timestamp + UNSTAKE_LOCKUP
    );
}

/**
 * @notice Complete unstaking after lockup period
 */
function unstake() external {
    Node storage node = nodes[msg.sender];
    require(node.unstakeRequestedAt != 0, "Unstake not requested");
    require(
        block.timestamp >= node.unstakeRequestedAt + UNSTAKE_LOCKUP,
        "Lockup period not elapsed"
    );

    uint256 returnAmount = node.stakedAmount;
    totalNetworkStorageGB -= node.storageCapacityGB;

    // Clear node data
    delete nodeIDToAddress[node.nodeID];
    delete nodes[msg.sender];

    // Return stake
    payable(msg.sender).transfer(returnAmount);

    emit NodeUnstaked(msg.sender, node.nodeID, returnAmount);
}

// --- Storage Contract Functions ---

/**
 * @notice Create a new storage contract for user data
 * @param merkleRoot Root hash of the data Merkle tree
 * @param dataSizeBytes Total size of data in bytes
 * @param durationDays How many days to store
 * @param assignedNodeIDs Array of 15 node IDs to store shards
 */
function createStorageContract(
    bytes32 merkleRoot,
    uint256 dataSizeBytes,
    uint256 durationDays,
    bytes32[] calldata assignedNodeIDs
) external payable returns (bytes32) {
    require(assignedNodeIDs.length == 15, "Must assign 15 nodes");
    require(dataSizeBytes > 0, "Invalid data size");
    require(durationDays > 0 && durationDays <= 365, "Invalid duration");

    // Calculate required payment
    uint256 requiredPayment = calculateStorageCost(
        dataSizeBytes,
        durationDays
    );
    require(msg.value >= requiredPayment, "Insufficient payment");

    // Verify all nodes are active and have capacity
    for (uint256 i = 0; i < 15; i++) {
        address nodeAddr = nodeIDToAddress[assignedNodeIDs[i]];
        require(nodeAddr != address(0), "Invalid node");
        require(nodes[nodeAddr].isActive, "Node not active");
    }

    // Generate contract ID
    bytes32 contractID = keccak256(abi.encodePacked(
        msg.sender,
        merkleRoot,
        block.timestamp,
        block.number
    ));

    // Create storage contract
    uint256 expiresBlock = block.number + (durationDays * 6400); // ~6400 blocks/day

    storageContracts[contractID] = StorageContract({
        contractID: contractID,
        user: msg.sender,
        merkleRoot: merkleRoot,
        dataSizeBytes: dataSizeBytes,
        shardCount: 15,
        durationBlocks: expiresBlock - block.number,
        rewardPoolTokens: msg.value,
        createdBlock: block.number,
        expiresBlock: expiresBlock,
        assignedNodeIDs: assignedNodeIDs,
        isActive: true
    });

    activeContracts.push(contractID);

    // Update node storage usage
    uint256 shardSize = dataSizeBytes / 10; // Approximate shard size
    for (uint256 i = 0; i < 15; i++) {
        address nodeAddr = nodeIDToAddress[assignedNodeIDs[i]];
        nodes[nodeAddr].storageUsedGB += shardSize / (1024 * 1024 * 1024);
    }

    emit StorageContractCreated(
        contractID,
        msg.sender,
        merkleRoot,
        dataSizeBytes,
        msg.value
    );

    return contractID;
}

/**
 * @notice Calculate storage cost
 * @param sizeBytes Size in bytes
 * @param durationDays Duration in days
 * @return cost in tokens
 */
function calculateStorageCost(
    uint256 sizeBytes,
    uint256 durationDays
) public pure returns (uint256) {
    // Cost formula: 0.1 tokens per GB per 30 days
    uint256 sizeGB = sizeBytes / (1024 * 1024 * 1024);
    if (sizeGB == 0) sizeGB = 1; // Minimum 1 GB

    uint256 cost = (sizeGB * durationDays * 10**17) / 30; // 0.1 tokens
    return cost;
}
```

### 2. ProofVerifier Contract

**Purpose**: Verify Merkle proofs for proof-of-storage challenges

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract ProofVerifier {

    // --- Events ---

    event ProofVerified(
        bytes32 indexed merkleRoot,
        bytes32 indexed leafHash,
        bool success
    );

    // --- Functions ---

    /**
     * @notice Verify a Merkle proof
     * @param proof Array of sibling hashes
     * @param root Expected Merkle root
     * @param leaf Leaf data to verify
     * @param index Position of leaf in tree
     * @return True if proof is valid
     */
    function verifyMerkleProof(
        bytes32[] calldata proof,
        bytes32 root,
        bytes32 leaf,
        uint256 index
    ) public pure returns (bool) {
        bytes32 computedHash = leaf;

        for (uint256 i = 0; i < proof.length; i++) {
            bytes32 proofElement = proof[i];

            if (index % 2 == 0) {
                // Current node is left child
                computedHash = keccak256(
                    abi.encodePacked(computedHash, proofElement)
                );
            } else {
                // Current node is right child
                computedHash = keccak256(
                    abi.encodePacked(proofElement, computedHash)
                );
            }

            index = index / 2;
        }

        return computedHash == root;
    }

    /**
     * @notice Verify proof with data instead of pre-hashed leaf
     * @param proof Merkle proof
     * @param root Expected root
     * @param data Original shard data
     * @param index Shard index
     * @return True if proof is valid
     */
    function verifyDataProof(
        bytes32[] calldata proof,
        bytes32 root,
        bytes calldata data,
        uint256 index
    ) external pure returns (bool) {
        bytes32 leaf = keccak256(data);
        return verifyMerkleProof(proof, root, leaf, index);
    }

    /**
     * @notice Batch verify multiple proofs (gas-efficient)
     * @param proofs Array of proof arrays
     * @param roots Array of expected roots
     * @param leaves Array of leaf hashes
     * @param indices Array of leaf indices
     * @return results Array of verification results
     */
    function batchVerifyProofs(
        bytes32[][] calldata proofs,
        bytes32[] calldata roots,
        bytes32[] calldata leaves,
        uint256[] calldata indices
    ) external pure returns (bool[] memory results) {
        require(
            proofs.length == roots.length &&
            roots.length == leaves.length &&
            leaves.length == indices.length,
            "Array length mismatch"
        );

        results = new bool[](proofs.length);

        for (uint256 i = 0; i < proofs.length; i++) {
            results[i] = verifyMerkleProof(
                proofs[i],
                roots[i],
                leaves[i],
                indices[i]
            );
        }

        return results;
    }
}
```

### 3. ChallengeManager Contract

**Purpose**: Issue and manage proof-of-storage challenges

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "./ProofVerifier.sol";
import "./StorageRewards.sol";

contract ChallengeManager {

    // --- State Variables ---

    StorageRewards public storageRewards;
    ProofVerifier public proofVerifier;

    struct Challenge {
        bytes32 challengeID;
        bytes32 contractID;
        bytes32 nodeID;
        uint256 shardIndex;
        bytes32 expectedRoot;
        uint256 issuedAt;
        uint256 deadline;
        bool isResolved;
        bool wasSuccessful;
    }

    mapping(bytes32 => Challenge) public challenges;
    mapping(bytes32 => bytes32[]) public nodeChallenges; // nodeID => challengeIDs

    uint256 public challengeCounter;

    // --- Events ---

    event ChallengeIssued(
        bytes32 indexed challengeID,
        bytes32 indexed nodeID,
        bytes32 contractID,
        uint256 shardIndex,
        uint256 deadline
    );

    event ChallengeResponseSubmitted(
        bytes32 indexed challengeID,
        bytes32 indexed nodeID,
        bool success,
        uint256 timestamp
    );

    event ChallengeExpired(
        bytes32 indexed challengeID,
        bytes32 indexed nodeID
    );

    // --- Constructor ---

    constructor(
        address _storageRewards,
        address _proofVerifier
    ) {
        storageRewards = StorageRewards(_storageRewards);
        proofVerifier = ProofVerifier(_proofVerifier);
    }

    // --- Functions ---

    /**
     * @notice Issue a random challenge to a node
     * @param contractID Storage contract to challenge
     * @param nodeID Node to challenge
     * @param shardIndex Which shard to prove
     */
    function issueChallenge(
        bytes32 contractID,
        bytes32 nodeID,
        uint256 shardIndex
    ) external returns (bytes32) {
        // Only contract owner or automated oracle can issue challenges
        require(
            msg.sender == address(storageRewards) ||
            hasRole(CHALLENGER_ROLE, msg.sender),
            "Not authorized"
        );

        // Get storage contract
        (
            ,
            ,
            bytes32 merkleRoot,
            ,
            ,
            ,
            ,
            ,
            uint256 expiresBlock,
            ,
            bool isActive
        ) = storageRewards.storageContracts(contractID);

        require(isActive, "Contract not active");
        require(block.number < expiresBlock, "Contract expired");
        require(shardIndex < 15, "Invalid shard index");

        // Generate challenge ID
        bytes32 challengeID = keccak256(abi.encodePacked(
            contractID,
            nodeID,
            shardIndex,
            block.timestamp,
            challengeCounter++
        ));

        // Create challenge
        challenges[challengeID] = Challenge({
            challengeID: challengeID,
            contractID: contractID,
            nodeID: nodeID,
            shardIndex: shardIndex,
            expectedRoot: merkleRoot,
            issuedAt: block.timestamp,
            deadline: block.timestamp + 5 minutes,
            isResolved: false,
            wasSuccessful: false
        });

        nodeChallenges[nodeID].push(challengeID);

        emit ChallengeIssued(
            challengeID,
            nodeID,
            contractID,
            shardIndex,
            block.timestamp + 5 minutes
        );

        return challengeID;
    }

    /**
     * @notice Submit response to a challenge
     * @param challengeID Challenge to respond to
     * @param shardData Actual shard data
     * @param merkleProof Proof that shard is part of Merkle tree
     */
    function respondToChallenge(
        bytes32 challengeID,
        bytes calldata shardData,
        bytes32[] calldata merkleProof
    ) external {
        Challenge storage challenge = challenges[challengeID];

        require(!challenge.isResolved, "Challenge already resolved");
        require(block.timestamp <= challenge.deadline, "Challenge expired");

        // Verify sender is the challenged node
        address nodeOwner = storageRewards.nodeIDToAddress(challenge.nodeID);
        require(msg.sender == nodeOwner, "Not the challenged node");

        // Verify Merkle proof
        bool isValid = proofVerifier.verifyDataProof(
            merkleProof,
            challenge.expectedRoot,
            shardData,
            challenge.shardIndex
        );

        // Mark as resolved
        challenge.isResolved = true;
        challenge.wasSuccessful = isValid;

        // Update node stats in StorageRewards
        if (isValid) {
            storageRewards.recordSuccessfulChallenge(challenge.nodeID);
        } else {
            storageRewards.recordFailedChallenge(challenge.nodeID);
        }

        emit ChallengeResponseSubmitted(
            challengeID,
            challenge.nodeID,
            isValid,
            block.timestamp
        );
    }

    /**
     * @notice Mark expired challenges as failed
     * @param challengeID Challenge to check
     */
    function expireChallenge(bytes32 challengeID) external {
        Challenge storage challenge = challenges[challengeID];

        require(!challenge.isResolved, "Already resolved");
        require(block.timestamp > challenge.deadline, "Not expired yet");

        // Mark as failed
        challenge.isResolved = true;
        challenge.wasSuccessful = false;

        // Slash node
        storageRewards.recordFailedChallenge(challenge.nodeID);

        emit ChallengeExpired(challengeID, challenge.nodeID);
    }

    /**
     * @notice Get all challenges for a node
     * @param nodeID Node to query
     * @return Array of challenge IDs
     */
    function getNodeChallenges(bytes32 nodeID)
        external
        view
        returns (bytes32[] memory)
    {
        return nodeChallenges[nodeID];
    }
}
```

---

## Merkle Proof System

### Merkle Tree Structure

```
Data Structure for 15 Shards:

Level 4 (Root):           [Root Hash]
                          /          \
Level 3:          [Hash 0-7]      [Hash 8-14]
                   /      \         /      \
Level 2:     [H 0-3]  [H 4-7]  [H 8-11] [H 12-14]
              /  \      /  \      /   \      /  \
Level 1:   [H0][H1] [H2][H3] [H4][H5] [H6][H7]
           ...

Level 0:   S0  S1  S2  S3  S4  S5  S6  S7  S8  S9  S10 S11 S12 S13 S14
          (Shard hashes)
```

### Go Implementation

```go
// pkg/meshstorage/merkle.go

package meshstorage

import (
    "crypto/sha256"
    "fmt"
)

// MerkleTree represents a Merkle tree for proof-of-storage
type MerkleTree struct {
    root   *MerkleNode
    leaves []*MerkleNode
}

// MerkleNode represents a node in the Merkle tree
type MerkleNode struct {
    Hash  [32]byte
    Left  *MerkleNode
    Right *MerkleNode
    Data  []byte // Only set for leaf nodes
    Index int    // Position in original data array
}

// MerkleProof represents a proof that a leaf is part of the tree
type MerkleProof struct {
    LeafHash   [32]byte   // Hash of the leaf data
    LeafIndex  int        // Position of leaf in tree
    Siblings   [][32]byte // Array of sibling hashes
    Root       [32]byte   // Expected root hash
}

// NewMerkleTree creates a Merkle tree from shard data
func NewMerkleTree(shards [][]byte) (*MerkleTree, error) {
    if len(shards) == 0 {
        return nil, fmt.Errorf("no shards provided")
    }

    // Create leaf nodes
    leaves := make([]*MerkleNode, len(shards))
    for i, shard := range shards {
        hash := sha256.Sum256(shard)
        leaves[i] = &MerkleNode{
            Hash:  hash,
            Data:  shard,
            Index: i,
        }
    }

    // Build tree bottom-up
    root := buildTree(leaves)

    return &MerkleTree{
        root:   root,
        leaves: leaves,
    }, nil
}

// buildTree recursively builds the Merkle tree
func buildTree(nodes []*MerkleNode) *MerkleNode {
    if len(nodes) == 1 {
        return nodes[0]
    }

    // Create parent level
    var parents []*MerkleNode

    for i := 0; i < len(nodes); i += 2 {
        left := nodes[i]
        var right *MerkleNode

        if i+1 < len(nodes) {
            right = nodes[i+1]
        } else {
            // Odd number of nodes, duplicate last node
            right = left
        }

        // Combine hashes
        combined := append(left.Hash[:], right.Hash[:]...)
        parentHash := sha256.Sum256(combined)

        parent := &MerkleNode{
            Hash:  parentHash,
            Left:  left,
            Right: right,
        }

        parents = append(parents, parent)
    }

    return buildTree(parents)
}

// Root returns the root hash of the tree
func (mt *MerkleTree) Root() [32]byte {
    return mt.root.Hash
}

// GenerateProof creates a Merkle proof for a specific shard index
func (mt *MerkleTree) GenerateProof(shardIndex int) (*MerkleProof, error) {
    if shardIndex < 0 || shardIndex >= len(mt.leaves) {
        return nil, fmt.Errorf("invalid shard index: %d", shardIndex)
    }

    leaf := mt.leaves[shardIndex]
    var siblings [][32]byte

    // Traverse from leaf to root, collecting sibling hashes
    currentNode := leaf
    currentIndex := shardIndex

    for currentNode != mt.root {
        // Find parent
        parent := findParent(mt.root, currentNode)
        if parent == nil {
            return nil, fmt.Errorf("parent not found for node")
        }

        // Add sibling hash
        if parent.Left == currentNode {
            siblings = append(siblings, parent.Right.Hash)
        } else {
            siblings = append(siblings, parent.Left.Hash)
        }

        currentNode = parent
        currentIndex = currentIndex / 2
    }

    return &MerkleProof{
        LeafHash:  leaf.Hash,
        LeafIndex: shardIndex,
        Siblings:  siblings,
        Root:      mt.root.Hash,
    }, nil
}

// findParent finds the parent node of a given node
func findParent(root, target *MerkleNode) *MerkleNode {
    if root == nil || root == target {
        return nil
    }

    if root.Left == target || root.Right == target {
        return root
    }

    if parent := findParent(root.Left, target); parent != nil {
        return parent
    }

    return findParent(root.Right, target)
}

// VerifyProof verifies a Merkle proof
func VerifyProof(proof *MerkleProof, shardData []byte) bool {
    // Hash the shard data
    computedHash := sha256.Sum256(shardData)

    // Verify leaf hash matches
    if computedHash != proof.LeafHash {
        return false
    }

    // Reconstruct path to root
    currentHash := computedHash
    currentIndex := proof.LeafIndex

    for _, sibling := range proof.Siblings {
        var combined []byte

        if currentIndex%2 == 0 {
            // Current node is left child
            combined = append(currentHash[:], sibling[:]...)
        } else {
            // Current node is right child
            combined = append(sibling[:], currentHash[:]...)
        }

        currentHash = sha256.Sum256(combined)
        currentIndex = currentIndex / 2
    }

    // Verify computed root matches expected root
    return currentHash == proof.Root
}

// SerializeProof converts proof to bytes for transmission
func (p *MerkleProof) SerializeProof() []byte {
    // Format: leafHash(32) + leafIndex(4) + siblingCount(2) + siblings(32*N) + root(32)
    size := 32 + 4 + 2 + (32 * len(p.Siblings)) + 32
    result := make([]byte, 0, size)

    // Leaf hash
    result = append(result, p.LeafHash[:]...)

    // Leaf index (4 bytes big-endian)
    result = append(result, byte(p.LeafIndex>>24))
    result = append(result, byte(p.LeafIndex>>16))
    result = append(result, byte(p.LeafIndex>>8))
    result = append(result, byte(p.LeafIndex))

    // Sibling count (2 bytes)
    sibCount := len(p.Siblings)
    result = append(result, byte(sibCount>>8))
    result = append(result, byte(sibCount))

    // Siblings
    for _, sib := range p.Siblings {
        result = append(result, sib[:]...)
    }

    // Root
    result = append(result, p.Root[:]...)

    return result
}
```

### Proof Size Analysis

For a tree with 15 shards:
```
Tree height: ceil(log2(15)) = 4 levels
Proof components:
- Leaf hash: 32 bytes
- Leaf index: 4 bytes
- Sibling count: 2 bytes
- Siblings: 4 × 32 = 128 bytes (4 siblings for height 4)
- Root: 32 bytes

Total: 32 + 4 + 2 + 128 + 32 = 198 bytes per proof
```

Gas cost analysis:
```
Calldata cost: 198 bytes × 16 gas/byte = 3,168 gas
Verification: ~5,000-10,000 gas
Total: ~13,000-15,000 gas per challenge

At 20 gwei gas price: ~$0.0003 per challenge
```

---

## Challenge-Response Protocol

### Challenge Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                   Challenge Lifecycle                        │
└─────────────────────────────────────────────────────────────┘

1. Challenge Generation (Daily, automated)
   ├─> Smart contract uses VRF for randomness
   ├─> Selects random shard from random storage contract
   ├─> Emits ChallengeIssued event
   └─> Sets 5-minute deadline

2. Node Detection
   ├─> Node listens to blockchain events
   ├─> Detects ChallengeIssued event for its nodeID
   ├─> Parses: contractID, shardIndex, deadline
   └─> Starts proof generation

3. Proof Generation (< 5 minutes)
   ├─> Retrieve shard from SQLite: storage.GetChunk(shardKey, index)
   ├─> Retrieve Merkle proof from storage
   ├─> Sign proof with node's private key
   └─> Prepare response transaction

4. Proof Submission
   ├─> Submit tx to ChallengeManager.respondToChallenge()
   ├─> Include: shardData, merkleProof
   ├─> Pay gas fee
   └─> Wait for confirmation

5. Verification (On-chain)
   ├─> ProofVerifier.verifyDataProof()
   ├─> Hash shard data
   ├─> Verify Merkle path
   ├─> Compare with stored root hash
   └─> Return success/failure

6. Result Processing
   ├─> Success:
   │   ├─> Increment successfulChallenges counter
   │   ├─> Add reputation bonus
   │   ├─> Emit ChallengeResponseSubmitted(true)
   │   └─> Earn challenge bonus tokens
   └─> Failure:
       ├─> Increment failedChallenges counter
       ├─> Slash 10% of stake
       ├─> Reduce reputation score
       ├─> Emit ChallengeResponseSubmitted(false)
       └─> Trigger re-replication

7. Timeout Handling (if no response)
   ├─> After 5 minutes, anyone can call expireChallenge()
   ├─> Automatically treated as failed challenge
   ├─> Node is slashed
   └─> Re-replication triggered
```

### Challenge Parameters

```go
// pkg/meshstorage/challenge.go

type ChallengeConfig struct {
    // How often to challenge each shard
    ChallengeIntervalMin time.Duration // 24 hours
    ChallengeIntervalMax time.Duration // 48 hours

    // Response deadline
    ResponseTimeout time.Duration // 5 minutes

    // Penalties
    SlashPercentage     uint8   // 10% of stake
    ReputationPenalty   uint16  // -500 points (out of 10000)

    // Bonuses
    ChallengeBonus      uint256 // 1 token per successful challenge
    PerfectMonthBonus   uint256 // 100 tokens for 30 days, 100% success

    // Thresholds
    MinSuccessRate      float64 // 95% required
    AutoKickThreshold   float64 // 80% - below this, auto-unstake
}

type Challenge struct {
    ChallengeID   [32]byte    // Unique challenge ID
    ContractID    [32]byte    // Which storage contract
    NodeID        [32]byte    // Which node to challenge
    ShardIndex    uint8       // Which shard (0-14)
    IssuedAt      time.Time   // When challenge was issued
    Deadline      time.Time   // Response deadline
    ExpectedRoot  [32]byte    // Merkle root to verify against
    Status        ChallengeStatus
}

type ChallengeStatus uint8

const (
    ChallengePending ChallengeStatus = iota
    ChallengeSuccess
    ChallengeFailed
    ChallengeExpired
)

type ChallengeResponse struct {
    ChallengeID   [32]byte      // Which challenge
    NodeID        [32]byte      // Responding node
    ShardData     []byte        // Actual shard data
    MerkleProof   *MerkleProof  // Proof of inclusion
    Signature     []byte        // Node's signature
    SubmittedAt   time.Time     // When submitted
}
```

### Node-side Challenge Handler

```go
// pkg/meshstorage/challenge_handler.go

package meshstorage

import (
    "context"
    "fmt"
    "time"

    "github.com/zentalk/protocol/pkg/blockchain"
)

// ChallengeHandler manages challenge responses for a storage node
type ChallengeHandler struct {
    node       *DHTNode
    blockchain *blockchain.Client
    storage    *LocalStorage
    signer     *blockchain.Signer
}

// NewChallengeHandler creates a new challenge handler
func NewChallengeHandler(
    node *DHTNode,
    bc *blockchain.Client,
    storage *LocalStorage,
    signer *blockchain.Signer,
) *ChallengeHandler {
    return &ChallengeHandler{
        node:       node,
        blockchain: bc,
        storage:    storage,
        signer:     signer,
    }
}

// Start begins listening for challenges
func (ch *ChallengeHandler) Start(ctx context.Context) error {
    // Subscribe to ChallengeIssued events
    challengeChan := make(chan *blockchain.ChallengeIssuedEvent)

    sub, err := ch.blockchain.SubscribeChallengeIssued(ctx, ch.node.ID(), challengeChan)
    if err != nil {
        return fmt.Errorf("failed to subscribe to challenges: %w", err)
    }
    defer sub.Unsubscribe()

    fmt.Printf("✅ Listening for challenges for node %s\n", ch.node.ID())

    for {
        select {
        case <-ctx.Done():
            return nil

        case event := <-challengeChan:
            // Handle challenge in background
            go ch.handleChallenge(ctx, event)

        case err := <-sub.Err():
            return fmt.Errorf("subscription error: %w", err)
        }
    }
}

// handleChallenge responds to a specific challenge
func (ch *ChallengeHandler) handleChallenge(
    ctx context.Context,
    event *blockchain.ChallengeIssuedEvent,
) {
    fmt.Printf("📝 Received challenge: %s (shard %d)\n",
        event.ChallengeID.Hex(),
        event.ShardIndex,
    )

    // Create timeout context
    deadline := time.Unix(int64(event.Deadline), 0)
    timeoutCtx, cancel := context.WithDeadline(ctx, deadline.Add(-30*time.Second))
    defer cancel()

    // Generate and submit proof
    err := ch.respondToChallenge(timeoutCtx, event)
    if err != nil {
        fmt.Printf("❌ Failed to respond to challenge: %v\n", err)
        // Log for later investigation
        ch.logFailedChallenge(event, err)
        return
    }

    fmt.Printf("✅ Successfully responded to challenge %s\n", event.ChallengeID.Hex())
}

// respondToChallenge generates and submits a proof
func (ch *ChallengeHandler) respondToChallenge(
    ctx context.Context,
    event *blockchain.ChallengeIssuedEvent,
) error {
    // 1. Retrieve shard from storage
    shardKey := generateShardKey(event.ContractID, event.ShardIndex)
    shardData, err := ch.storage.GetChunk(shardKey, 0)
    if err != nil {
        return fmt.Errorf("shard not found in storage: %w", err)
    }

    // 2. Retrieve Merkle proof from storage metadata
    // (Stored when shard was originally received)
    proofData, err := ch.storage.GetMetadata(shardKey, "merkle_proof")
    if err != nil {
        return fmt.Errorf("merkle proof not found: %w", err)
    }

    proof, err := DeserializeMerkleProof(proofData)
    if err != nil {
        return fmt.Errorf("failed to deserialize proof: %w", err)
    }

    // 3. Verify proof locally before submitting (sanity check)
    if !VerifyProof(proof, shardData) {
        return fmt.Errorf("local proof verification failed - data corrupted")
    }

    // 4. Prepare response
    response := &ChallengeResponse{
        ChallengeID: event.ChallengeID,
        NodeID:      nodeIDToBytes32(ch.node.ID()),
        ShardData:   shardData,
        MerkleProof: proof,
        SubmittedAt: time.Now(),
    }

    // 5. Sign response
    signature, err := ch.signer.Sign(response.Hash())
    if err != nil {
        return fmt.Errorf("failed to sign response: %w", err)
    }
    response.Signature = signature

    // 6. Submit to blockchain
    tx, err := ch.blockchain.RespondToChallenge(ctx, response)
    if err != nil {
        return fmt.Errorf("failed to submit response: %w", err)
    }

    // 7. Wait for confirmation
    receipt, err := ch.blockchain.WaitForReceipt(ctx, tx.Hash())
    if err != nil {
        return fmt.Errorf("transaction failed: %w", err)
    }

    if receipt.Status != 1 {
        return fmt.Errorf("transaction reverted")
    }

    return nil
}

// generateShardKey creates a storage key for a shard
func generateShardKey(contractID [32]byte, shardIndex uint8) string {
    return fmt.Sprintf("contract_%s_shard_%d",
        contractID.Hex(),
        shardIndex,
    )
}

// logFailedChallenge records failed challenges for analysis
func (ch *ChallengeHandler) logFailedChallenge(
    event *blockchain.ChallengeIssuedEvent,
    err error,
) {
    // Store in local database for investigation
    failureLog := struct {
        ChallengeID [32]byte
        ContractID  [32]byte
        ShardIndex  uint8
        Error       string
        Timestamp   time.Time
    }{
        ChallengeID: event.ChallengeID,
        ContractID:  event.ContractID,
        ShardIndex:  event.ShardIndex,
        Error:       err.Error(),
        Timestamp:   time.Now(),
    }

    // Store in SQLite for later review
    ch.storage.LogChallengeFailure(failureLog)
}
```

---

## Token Economics

### Token Supply & Distribution

```
Total Token Supply: 1,000,000,000 ZENT

Distribution:
├─ 40% (400M) - Storage Node Rewards Pool (vested over 10 years)
├─ 25% (250M) - Team & Development (4-year vesting)
├─ 20% (200M) - Community & Ecosystem Growth
├─ 10% (100M) - Initial Liquidity
└─  5% ( 50M) - Reserve Fund

Daily Emission Schedule:
Year 1: 109,589 ZENT/day (40M / 365)
Year 2:  82,192 ZENT/day (30M / 365) (-25%)
Year 3:  61,644 ZENT/day (22.5M / 365) (-25%)
Year 4:  46,233 ZENT/day (16.9M / 365) (-25%)
...
Year 10:  5,479 ZENT/day (2M / 365)
```

### Reward Formula

```
Node Daily Reward = Base Reward + Performance Bonus + Challenge Bonus

Where:

Base Reward =
    (Node Storage GB / Total Network GB) ×
    Daily Pool ×
    40% (storage weight)

Performance Bonus =
    (Node Uptime % / 100) ×
    Daily Pool ×
    30% (uptime weight)

Challenge Bonus =
    (Successful Challenges / Total Challenges) ×
    Daily Pool ×
    20% (challenge weight) +
    Perfect Month Bonus (if 100% success rate)

Longevity Bonus =
    (Node Age Days / Max Age Days) ×
    Daily Pool ×
    10% (longevity weight)
```

### Example Reward Calculations

**Scenario 1: New Node (Month 1)**

Network state:
- Total network storage: 10,000 GB
- Total nodes: 100
- Daily reward pool: 109,589 ZENT

Node specs:
- Storage provided: 100 GB (1% of network)
- Uptime: 99.5%
- Challenges: 30/30 successful (100%)
- Age: 30 days (max tracked: 365 days)

Calculation:
```
Base = (100 / 10000) × 109,589 × 0.40
     = 0.01 × 109,589 × 0.40
     = 438.36 ZENT

Performance = 0.995 × 109,589 × 0.30
            = 32,729.40 ZENT

Challenge = 1.0 × 109,589 × 0.20
          = 21,917.80 ZENT

Longevity = (30 / 365) × 109,589 × 0.10
          = 901.20 ZENT

Total Daily = 438.36 + 32,729.40 + 21,917.80 + 901.20
            = 55,986.76 ZENT/day

Monthly (30 days) = 55,986.76 × 30 = 1,679,603 ZENT

If ZENT = $0.001: $1,679.60/month
If ZENT = $0.01:  $16,796/month
```

**Scenario 2: Established Node (Year 2)**

Network state:
- Total network storage: 100,000 GB (10x growth)
- Total nodes: 500
- Daily reward pool: 82,192 ZENT (Year 2 emission)

Node specs:
- Storage: 500 GB (0.5% of network)
- Uptime: 99.9%
- Challenges: 360/365 successful (98.6%)
- Age: 365 days

Calculation:
```
Base = (500 / 100000) × 82,192 × 0.40
     = 164.38 ZENT

Performance = 0.999 × 82,192 × 0.30
            = 24,615.50 ZENT

Challenge = 0.986 × 82,192 × 0.20
          = 16,208.25 ZENT

Longevity = (365 / 365) × 82,192 × 0.10
          = 8,219.20 ZENT

Total Daily = 49,207.33 ZENT/day
Monthly = 1,476,220 ZENT

If ZENT = $0.01: $14,762/month
```

### Penalty System

```
Penalty Triggers:

1. Failed Challenge: -10% of stake
   Example: 1000 ZENT stake → 100 ZENT slashed

2. Extended Downtime:
   - < 1 hour: No penalty
   - 1-24 hours: -1% per hour of daily rewards
   - > 24 hours: -10% of stake + removal from active set

3. Data Corruption:
   - Detected during challenge
   - -50% of stake
   - Forced data re-replication at node's expense

4. Repeated Failures:
   - 3 failures in 30 days: -25% stake warning
   - 5 failures in 30 days: -50% stake + 30-day suspension
   - 10 failures in 30 days: Complete stake slashing + permanent ban

5. Fraudulent Behavior:
   - Fake proofs, signature manipulation
   - -100% of stake (complete slashing)
   - Permanent ban from network
   - Potential legal action
```

### Economics Simulation

```
Network Growth Projection (5 years):

Year 1:
- Nodes: 100 → 500
- Storage: 10 TB → 50 TB
- Users: 1,000 → 10,000
- Monthly costs per user: $1
- Monthly revenue: $10,000
- Monthly rewards distributed: $100,000 (token inflation)
- Token price: $0.001

Year 3:
- Nodes: 2,000
- Storage: 500 TB
- Users: 100,000
- Monthly costs per user: $1
- Monthly revenue: $100,000
- Monthly rewards: $50,000 (reduced emission)
- Token price: $0.01 (10x appreciation)

Year 5:
- Nodes: 10,000
- Storage: 5 PB
- Users: 1,000,000
- Monthly costs per user: $0.50 (economies of scale)
- Monthly revenue: $500,000
- Monthly rewards: $25,000 (reduced emission)
- Token price: $0.05 (50x from Year 1)

Break-even Analysis:
Revenue > Rewards when:
$100,000/month > Token Emissions
Occurs around Year 2-3 with 50K-100K users
```

---

## API Specifications

### Blockchain Client API (Go)

```go
// pkg/blockchain/client.go

package blockchain

import (
    "context"
    "math/big"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/ethclient"
)

// Client provides blockchain interaction for storage nodes
type Client struct {
    eth              *ethclient.Client
    storageRewards   *StorageRewardsContract
    challengeManager *ChallengeManagerContract
    proofVerifier    *ProofVerifierContract
    chainID          *big.Int
}

// NewClient creates a new blockchain client
func NewClient(rpcURL string, contracts ContractAddresses) (*Client, error) {
    eth, err := ethclient.Dial(rpcURL)
    if err != nil {
        return nil, err
    }

    chainID, err := eth.ChainID(context.Background())
    if err != nil {
        return nil, err
    }

    // Initialize contract bindings
    storageRewards, err := NewStorageRewardsContract(
        contracts.StorageRewards,
        eth,
    )
    if err != nil {
        return nil, err
    }

    challengeManager, err := NewChallengeManagerContract(
        contracts.ChallengeManager,
        eth,
    )
    if err != nil {
        return nil, err
    }

    proofVerifier, err := NewProofVerifierContract(
        contracts.ProofVerifier,
        eth,
    )
    if err != nil {
        return nil, err
    }

    return &Client{
        eth:              eth,
        storageRewards:   storageRewards,
        challengeManager: challengeManager,
        proofVerifier:    proofVerifier,
        chainID:          chainID,
    }, nil
}

// RegisterNode registers a new storage node
func (c *Client) RegisterNode(
    ctx context.Context,
    nodeID [32]byte,
    storageCapacityGB *big.Int,
    stakeAmount *big.Int,
    privateKey *ecdsa.PrivateKey,
) (*types.Transaction, error) {
    auth, err := bind.NewKeyedTransactorWithChainID(privateKey, c.chainID)
    if err != nil {
        return nil, err
    }

    auth.Value = stakeAmount

    return c.storageRewards.RegisterNode(auth, nodeID, storageCapacityGB)
}

// CreateStorageContract creates a new storage contract
func (c *Client) CreateStorageContract(
    ctx context.Context,
    params *StorageContractParams,
    privateKey *ecdsa.PrivateKey,
) (*types.Transaction, error) {
    auth, err := bind.NewKeyedTransactorWithChainID(privateKey, c.chainID)
    if err != nil {
        return nil, err
    }

    auth.Value = params.Payment

    return c.storageRewards.CreateStorageContract(
        auth,
        params.MerkleRoot,
        params.DataSize,
        params.DurationDays,
        params.AssignedNodes,
    )
}

// RespondToChallenge submits a challenge response
func (c *Client) RespondToChallenge(
    ctx context.Context,
    response *ChallengeResponse,
    privateKey *ecdsa.PrivateKey,
) (*types.Transaction, error) {
    auth, err := bind.NewKeyedTransactorWithChainID(privateKey, c.chainID)
    if err != nil {
        return nil, err
    }

    // Convert MerkleProof to bytes32 array
    siblings := make([][32]byte, len(response.MerkleProof.Siblings))
    for i, sib := range response.MerkleProof.Siblings {
        siblings[i] = sib
    }

    return c.challengeManager.RespondToChallenge(
        auth,
        response.ChallengeID,
        response.ShardData,
        siblings,
    )
}

// SubscribeChallengeIssued subscribes to challenge events
func (c *Client) SubscribeChallengeIssued(
    ctx context.Context,
    nodeID peer.ID,
    ch chan<- *ChallengeIssuedEvent,
) (event.Subscription, error) {
    nodeIDBytes := nodeIDToBytes32(nodeID)

    opts := &bind.WatchOpts{
        Context: ctx,
    }

    return c.challengeManager.WatchChallengeIssued(
        opts,
        ch,
        [][32]byte{nodeIDBytes}, // Filter by nodeID
    )
}

// GetNodeInfo retrieves node information from blockchain
func (c *Client) GetNodeInfo(
    ctx context.Context,
    nodeAddress common.Address,
) (*NodeInfo, error) {
    node, err := c.storageRewards.Nodes(&bind.CallOpts{Context: ctx}, nodeAddress)
    if err != nil {
        return nil, err
    }

    return &NodeInfo{
        Owner:               node.Owner,
        NodeID:              node.NodeID,
        StakedAmount:        node.StakedAmount,
        StorageCapacityGB:   node.StorageCapacityGB,
        StorageUsedGB:       node.StorageUsedGB,
        SuccessfulChallenges: node.SuccessfulChallenges,
        FailedChallenges:    node.FailedChallenges,
        TotalEarnedRewards:  node.TotalEarnedRewards,
        ReputationScore:     node.ReputationScore,
        IsActive:            node.IsActive,
    }, nil
}

// ClaimRewards claims accumulated rewards
func (c *Client) ClaimRewards(
    ctx context.Context,
    privateKey *ecdsa.PrivateKey,
) (*types.Transaction, error) {
    auth, err := bind.NewKeyedTransactorWithChainID(privateKey, c.chainID)
    if err != nil {
        return nil, err
    }

    return c.storageRewards.ClaimRewards(auth)
}
```

### REST API for Users (Express/Node.js)

```typescript
// api/routes/storage.ts

import express from 'express';
import { StorageService } from '../services/storage';
import { BlockchainService } from '../services/blockchain';

const router = express.Router();

/**
 * POST /api/storage/upload
 * Upload data to mesh storage network
 */
router.post('/upload', async (req, res) => {
  try {
    const { userId, data, durationDays } = req.body;

    // 1. Encrypt data (E2EE)
    const encrypted = await encryptData(data, userId);

    // 2. Erasure encode (10+5)
    const shards = await erasureEncode(encrypted);

    // 3. Generate Merkle tree
    const merkleTree = generateMerkleTree(shards);
    const merkleRoot = merkleTree.root();

    // 4. Find storage nodes via DHT
    const nodes = await findStorageNodes(15);

    // 5. Calculate cost
    const cost = calculateStorageCost(encrypted.length, durationDays);

    // 6. Create blockchain storage contract
    const tx = await blockchainService.createStorageContract({
      merkleRoot,
      dataSize: encrypted.length,
      durationDays,
      assignedNodes: nodes.map(n => n.id),
      payment: cost,
    });

    // 7. Distribute shards to nodes
    const distribution = await distributeShards(shards, nodes, merkleTree);

    // 8. Return result
    res.json({
      success: true,
      contractId: tx.hash,
      merkleRoot: merkleRoot.toString('hex'),
      shardCount: 15,
      cost: cost.toString(),
      nodes: nodes.map(n => ({ id: n.id, address: n.address })),
    });

  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

/**
 * GET /api/storage/retrieve/:contractId
 * Retrieve data from mesh storage
 */
router.get('/retrieve/:contractId', async (req, res) => {
  try {
    const { contractId } = req.params;
    const { userId } = req.query;

    // 1. Get storage contract from blockchain
    const contract = await blockchainService.getStorageContract(contractId);

    // 2. Retrieve shards from nodes
    const shards = await retrieveShards(contract.assignedNodes, contractId);

    // 3. Verify shards using Merkle proofs
    const verified = verifyShards(shards, contract.merkleRoot);
    if (!verified) {
      throw new Error('Shard verification failed');
    }

    // 4. Decode erasure coded data
    const encrypted = await erasureDecode(shards);

    // 5. Decrypt data
    const data = await decryptData(encrypted, userId);

    // 6. Return data
    res.json({
      success: true,
      data,
      contractId,
    });

  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

/**
 * GET /api/storage/status/:contractId
 * Get status of stored data
 */
router.get('/status/:contractId', async (req, res) => {
  try {
    const { contractId } = req.params;

    // Get contract from blockchain
    const contract = await blockchainService.getStorageContract(contractId);

    // Check shard availability
    const shardStatus = await checkShardAvailability(contract.assignedNodes);

    // Calculate health score
    const health = calculateHealthScore(shardStatus);

    res.json({
      success: true,
      contractId,
      status: contract.isActive ? 'active' : 'expired',
      health: health.percentage,
      availableShards: shardStatus.available,
      totalShards: 15,
      expiresAt: new Date(contract.expiresBlock * 12 * 1000),
    });

  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

export default router;
```

---

## Integration with Phase 2

### Updated Distributed Storage

```go
// pkg/meshstorage/distributed.go (Updated)

// StoreDistributed now includes blockchain integration
func (ds *DistributedStorage) StoreDistributed(
    ctx context.Context,
    userAddr string,
    chunkID int,
    data []byte,
) (*DistributedChunk, error) {
    // Phase 2 logic (already implemented)
    encoded, err := ds.encoder.Encode(data)
    if err != nil {
        return nil, fmt.Errorf("failed to encode data: %w", err)
    }

    key := generateStorageKey(userAddr, chunkID)
    targetPeers, err := ds.findStorageNodes(ctx, key, TotalShards)
    if err != nil {
        return nil, fmt.Errorf("failed to find storage nodes: %w", err)
    }

    // NEW: Phase 3 blockchain integration

    // 1. Generate Merkle tree
    merkleTree, err := NewMerkleTree(encoded.Shards)
    if err != nil {
        return nil, fmt.Errorf("failed to create merkle tree: %w", err)
    }

    // 2. Create storage contract on blockchain
    contractParams := &blockchain.StorageContractParams{
        User:          userAddr,
        MerkleRoot:    merkleTree.Root(),
        DataSize:      big.NewInt(int64(len(data))),
        DurationDays:  big.NewInt(30), // Default 30 days
        AssignedNodes: peerIDsToBytes32(targetPeers),
        Payment:       calculatePayment(len(data), 30),
    }

    tx, err := ds.blockchain.CreateStorageContract(ctx, contractParams, ds.privateKey)
    if err != nil {
        return nil, fmt.Errorf("failed to create storage contract: %w", err)
    }

    // Wait for confirmation
    receipt, err := ds.blockchain.WaitForReceipt(ctx, tx.Hash())
    if err != nil {
        return nil, fmt.Errorf("contract creation failed: %w", err)
    }

    contractID := receipt.Logs[0].Topics[1] // Extract from event

    // 3. Distribute shards with Merkle proofs
    shardLocations := make([]ShardLocation, TotalShards)
    var wg sync.WaitGroup
    errChan := make(chan error, TotalShards)

    for i := 0; i < TotalShards; i++ {
        wg.Add(1)
        go func(shardIndex int) {
            defer wg.Done()

            targetPeer := targetPeers[shardIndex]

            // Generate Merkle proof for this shard
            proof, err := merkleTree.GenerateProof(shardIndex)
            if err != nil {
                errChan <- fmt.Errorf("proof generation failed: %w", err)
                return
            }

            // Store shard with proof
            if targetPeer == ds.node.ID() {
                // Local storage
                shardKey := fmt.Sprintf("%s_%d_shard_%d", userAddr, chunkID, shardIndex)
                err = ds.node.Storage().StoreChunk(shardKey, shardIndex, encoded.Shards[shardIndex])
                if err != nil {
                    errChan <- fmt.Errorf("local storage failed: %w", err)
                    return
                }
                // Store Merkle proof as metadata
                ds.node.Storage().StoreMetadata(shardKey, "merkle_proof", proof.SerializeProof())
            } else {
                // Remote storage via RPC
                _, err = ds.client.StoreShard(
                    ctx,
                    targetPeer,
                    generateShardKey(contractID, uint8(shardIndex)),
                    shardIndex,
                    encoded.Shards[shardIndex],
                    userAddr,
                    chunkID,
                )
                if err != nil {
                    errChan <- fmt.Errorf("remote storage failed: %w", err)
                    return
                }
            }

            // Record shard location
            addrs := ds.node.Host().Peerstore().Addrs(targetPeer)
            addrStrs := make([]string, len(addrs))
            for j, addr := range addrs {
                addrStrs[j] = addr.String()
            }

            shardLocations[shardIndex] = ShardLocation{
                ShardIndex: shardIndex,
                PeerID:     targetPeer,
                PeerAddrs:  addrStrs,
            }
        }(i)
    }

    wg.Wait()
    close(errChan)

    if len(errChan) > 0 {
        return nil, <-errChan
    }

    return &DistributedChunk{
        UserAddr:       userAddr,
        ChunkID:        chunkID,
        OriginalSize:   len(data),
        ShardSize:      encoded.ShardSize,
        ShardLocations: shardLocations,
        MerkleRoot:     merkleTree.Root(),     // NEW
        ContractID:     contractID,            // NEW
    }, nil
}
```

### Updated Storage Node Main

```go
// cmd/storage-node/main.go (Updated)

func main() {
    // Parse flags
    port := flag.Int("port", 9000, "Port to listen on")
    dataDir := flag.String("data", "./storage-data", "Data directory")
    bootstrap := flag.String("bootstrap", "", "Bootstrap node address")
    rpcURL := flag.String("rpc", "http://localhost:8545", "Blockchain RPC URL")
    privateKeyHex := flag.String("key", "", "Private key for signing transactions")
    stake := flag.String("stake", "1000", "Amount to stake (in tokens)")
    storageGB := flag.Int("storage", 100, "Storage capacity in GB")
    flag.Parse()

    // Initialize blockchain client
    contracts := blockchain.ContractAddresses{
        StorageRewards:   common.HexToAddress("0x..."),
        ChallengeManager: common.HexToAddress("0x..."),
        ProofVerifier:    common.HexToAddress("0x..."),
    }

    bc, err := blockchain.NewClient(*rpcURL, contracts)
    if err != nil {
        log.Fatalf("Failed to connect to blockchain: %v", err)
    }

    // Load private key
    privateKey, err := crypto.HexToECDSA(*privateKeyHex)
    if err != nil {
        log.Fatalf("Invalid private key: %v", err)
    }

    // Create DHT node
    node, err := meshstorage.NewDHTNode(*port, *dataDir)
    if err != nil {
        log.Fatalf("Failed to create node: %v", err)
    }

    // Register node on blockchain
    nodeIDBytes := nodeIDToBytes32(node.ID())
    stakeAmount, _ := new(big.Int).SetString(*stake, 10)
    stakeAmount = stakeAmount.Mul(stakeAmount, big.NewInt(1e18)) // Convert to wei

    fmt.Printf("Registering node on blockchain...\n")
    tx, err := bc.RegisterNode(
        context.Background(),
        nodeIDBytes,
        big.NewInt(int64(*storageGB)),
        stakeAmount,
        privateKey,
    )
    if err != nil {
        log.Fatalf("Failed to register node: %v", err)
    }

    fmt.Printf("Registration tx: %s\n", tx.Hash().Hex())
    fmt.Printf("Waiting for confirmation...\n")

    receipt, err := bc.WaitForReceipt(context.Background(), tx.Hash())
    if err != nil {
        log.Fatalf("Registration failed: %v", err)
    }

    fmt.Printf("✅ Node registered successfully!\n")
    fmt.Printf("Node ID: %s\n", node.ID())
    fmt.Printf("Staked: %s ZENT\n", stake)
    fmt.Printf("Storage: %d GB\n", *storageGB)

    // Start challenge handler
    signer := blockchain.NewSigner(privateKey)
    challengeHandler := meshstorage.NewChallengeHandler(
        node,
        bc,
        node.Storage(),
        signer,
    )

    go func() {
        if err := challengeHandler.Start(context.Background()); err != nil {
            log.Printf("Challenge handler error: %v", err)
        }
    }()

    // Start RPC handler
    rpcHandler := meshstorage.NewRPCHandler(node)
    rpcHandler.SetupStreamHandler()

    // Connect to bootstrap if provided
    if *bootstrap != "" {
        if err := node.Bootstrap(context.Background(), []string{*bootstrap}); err != nil {
            log.Fatalf("Failed to bootstrap: %v", err)
        }
    }

    fmt.Printf("🚀 Storage node running on port %d\n", *port)
    fmt.Printf("📊 Monitoring for challenges...\n")

    // Wait for interrupt
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, os.Interrupt, syscall.SIGTERM)
    <-sigCh

    fmt.Printf("\n Shutting down...\n")
    node.Close()
}
```

---

## Security Considerations

### 1. Smart Contract Security

**Vulnerabilities to Address:**

```solidity
// Reentrancy Protection
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract StorageRewards is ReentrancyGuard {
    function claimRewards() external nonReentrant {
        // Safe from reentrancy attacks
    }
}

// Integer Overflow Protection (Solidity 0.8+)
// Built-in overflow checks in 0.8+

// Access Control
import "@openzeppelin/contracts/access/AccessControl.sol";

contract StorageRewards is AccessControl {
    bytes32 public constant CHALLENGER_ROLE = keccak256("CHALLENGER");

    function issueChallenge() external onlyRole(CHALLENGER_ROLE) {
        // Only authorized challengers
    }
}

// Front-running Protection
// Use commit-reveal for sensitive operations
uint256 private commitDeadline;
mapping(address => bytes32) private commits;

function commitResponse(bytes32 commitment) external {
    commits[msg.sender] = commitment;
    commitDeadline = block.timestamp + 1 minutes;
}

function revealResponse(bytes32 nonce, bytes calldata data) external {
    require(block.timestamp >= commitDeadline, "Too early");
    require(
        keccak256(abi.encodePacked(nonce, data)) == commits[msg.sender],
        "Invalid reveal"
    );
    // Process response
}
```

**Audit Checklist:**

- [ ] Reentrancy guards on all state-changing functions
- [ ] Access control on administrative functions
- [ ] Integer overflow/underflow protection
- [ ] Front-running protection for challenges
- [ ] Gas optimization for loops and storage
- [ ] Event emission for all state changes
- [ ] Proper error handling and reverts
- [ ] Time-based logic using block.timestamp safely
- [ ] External call safety (checks-effects-interactions)
- [ ] Upgrade mechanism (if using proxies)

### 2. Merkle Proof Security

**Attack Vectors:**

1. **Second Preimage Attack**
   - Attacker tries to find different data with same hash
   - Mitigation: Use SHA256 (collision-resistant)

2. **Proof Forgery**
   - Attacker tries to construct valid proof for fake data
   - Mitigation: Require data + proof, not just proof

3. **Replay Attack**
   - Reuse old proof for new challenge
   - Mitigation: Include challenge ID and timestamp in verification

**Secure Implementation:**

```go
// Include challenge context in verification
func VerifyProofWithContext(
    proof *MerkleProof,
    shardData []byte,
    challengeID [32]byte,
    timestamp uint64,
) bool {
    // Compute leaf hash with context
    contextData := append(shardData, challengeID[:]...)
    contextData = append(contextData, timestampToBytes(timestamp)...)

    leafHash := sha256.Sum256(contextData)

    // Verify proof normally
    return verifyMerkleProofInternal(proof, leafHash)
}
```

### 3. Network Security

**DDoS Protection:**

```go
// Rate limiting for RPC requests
type RateLimiter struct {
    requests map[peer.ID]*RequestCounter
    limit    int
    window   time.Duration
}

func (rl *RateLimiter) Allow(peerID peer.ID) bool {
    counter := rl.requests[peerID]
    if counter == nil {
        counter = &RequestCounter{
            count:     1,
            resetTime: time.Now().Add(rl.window),
        }
        rl.requests[peerID] = counter
        return true
    }

    if time.Now().After(counter.resetTime) {
        counter.count = 1
        counter.resetTime = time.Now().Add(rl.window)
        return true
    }

    if counter.count >= rl.limit {
        return false // Rate limit exceeded
    }

    counter.count++
    return true
}

// Use in RPC handler
func (h *RPCHandler) handleStream(stream network.Stream) {
    peerID := stream.Conn().RemotePeer()

    if !h.rateLimiter.Allow(peerID) {
        stream.Reset()
        return
    }

    // Process request
}
```

**Eclipse Attack Protection:**

```go
// Prevent attacker from isolating node
func (n *DHTNode) Bootstrap(ctx context.Context, peers []string) error {
    // Connect to multiple bootstrap nodes
    if len(peers) < 3 {
        return fmt.Errorf("need at least 3 bootstrap peers")
    }

    // Verify diversity (different /24 subnets)
    if !verifyPeerDiversity(peers) {
        return fmt.Errorf("bootstrap peers not diverse enough")
    }

    // Continue bootstrapping
}
```

### 4. Data Integrity

**Corruption Detection:**

```go
// Periodic self-verification
func (n *DHTNode) VerifyStorageIntegrity(ctx context.Context) error {
    chunks, err := n.storage.ListAllChunks()
    if err != nil {
        return err
    }

    corruptedCount := 0

    for _, chunk := range chunks {
        // Retrieve Merkle proof
        proofData, err := n.storage.GetMetadata(chunk.UserAddr, "merkle_proof")
        if err != nil {
            log.Printf("Missing proof for chunk %s", chunk.UserAddr)
            continue
        }

        proof, err := DeserializeMerkleProof(proofData)
        if err != nil {
            log.Printf("Invalid proof format for %s", chunk.UserAddr)
            continue
        }

        // Verify data integrity
        if !VerifyProof(proof, chunk.Data) {
            log.Printf("❌ Data corruption detected: %s", chunk.UserAddr)
            corruptedCount++

            // Alert and trigger re-download
            n.alertDataCorruption(chunk.UserAddr)
        }
    }

    if corruptedCount > 0 {
        return fmt.Errorf("found %d corrupted chunks", corruptedCount)
    }

    return nil
}
```

---

## Testing Strategy

### Unit Tests

```go
// pkg/meshstorage/merkle_test.go

func TestMerkleTreeCreation(t *testing.T) {
    shards := generateTestShards(15)

    tree, err := NewMerkleTree(shards)
    assert.NoError(t, err)
    assert.NotNil(t, tree)
    assert.Len(t, tree.leaves, 15)
}

func TestMerkleProofGeneration(t *testing.T) {
    shards := generateTestShards(15)
    tree, _ := NewMerkleTree(shards)

    for i := 0; i < 15; i++ {
        proof, err := tree.GenerateProof(i)
        assert.NoError(t, err)
        assert.NotNil(t, proof)
        assert.Equal(t, i, proof.LeafIndex)
    }
}

func TestMerkleProofVerification(t *testing.T) {
    shards := generateTestShards(15)
    tree, _ := NewMerkleTree(shards)

    // Test valid proofs
    for i := 0; i < 15; i++ {
        proof, _ := tree.GenerateProof(i)
        valid := VerifyProof(proof, shards[i])
        assert.True(t, valid, "Proof for shard %d should be valid", i)
    }

    // Test invalid proof (modified data)
    proof, _ := tree.GenerateProof(0)
    modifiedData := append([]byte("corrupted"), shards[0]...)
    valid := VerifyProof(proof, modifiedData)
    assert.False(t, valid, "Proof should fail for modified data")
}

func TestMerkleProofSerialization(t *testing.T) {
    shards := generateTestShards(15)
    tree, _ := NewMerkleTree(shards)

    proof, _ := tree.GenerateProof(7)

    // Serialize
    serialized := proof.SerializeProof()
    assert.NotEmpty(t, serialized)

    // Deserialize
    deserialized, err := DeserializeMerkleProof(serialized)
    assert.NoError(t, err)
    assert.Equal(t, proof.LeafHash, deserialized.LeafHash)
    assert.Equal(t, proof.LeafIndex, deserialized.LeafIndex)
    assert.Equal(t, proof.Root, deserialized.Root)
}

// pkg/blockchain/client_test.go

func TestRegisterNode(t *testing.T) {
    // Use Ganache or Hardhat for local testing
    client, _ := NewClient("http://localhost:8545", testContracts)

    nodeID := [32]byte{1, 2, 3} // Test node ID
    stake := big.NewInt(1000e18) // 1000 tokens
    capacity := big.NewInt(100)  // 100 GB

    tx, err := client.RegisterNode(
        context.Background(),
        nodeID,
        capacity,
        stake,
        testPrivateKey,
    )

    assert.NoError(t, err)
    assert.NotNil(t, tx)

    // Wait for confirmation
    receipt, _ := client.WaitForReceipt(context.Background(), tx.Hash())
    assert.Equal(t, uint64(1), receipt.Status)
}

func TestChallengeResponse(t *testing.T) {
    // Setup: Register node, create storage contract
    // ...

    // Issue challenge
    challengeTx, _ := client.IssueChallenge(
        context.Background(),
        contractID,
        nodeID,
        5, // Shard index
        testPrivateKey,
    )

    // Generate proof
    shardData := testShards[5]
    merkleTree, _ := NewMerkleTree(testShards)
    proof, _ := merkleTree.GenerateProof(5)

    // Respond to challenge
    response := &ChallengeResponse{
        ChallengeID: challengeID,
        NodeID:      nodeID,
        ShardData:   shardData,
        MerkleProof: proof,
    }

    responseTx, err := client.RespondToChallenge(
        context.Background(),
        response,
        testPrivateKey,
    )

    assert.NoError(t, err)

    // Verify challenge resolved successfully
    receipt, _ := client.WaitForReceipt(context.Background(), responseTx.Hash())
    assert.Equal(t, uint64(1), receipt.Status)
}
```

### Integration Tests

```go
// pkg/meshstorage/integration_test.go

func TestEndToEndStorageWithBlockchain(t *testing.T) {
    // 1. Setup local blockchain (Ganache)
    blockchain := setupLocalBlockchain(t)
    defer blockchain.Cleanup()

    // 2. Deploy contracts
    contracts := deployContracts(t, blockchain)

    // 3. Create 3 storage nodes
    nodes := make([]*DHTNode, 3)
    for i := 0; i < 3; i++ {
        node, _ := NewDHTNode(9000+i, fmt.Sprintf("./test-data-%d", i))
        nodes[i] = node

        // Register node on blockchain
        registerNode(t, node, blockchain, contracts)
    }

    // 4. Connect nodes
    bootstrapNodes(t, nodes)

    // 5. Store data
    testData := []byte("Test data for Phase 3 integration")

    ds := nodes[0].DistributedStorage()
    chunk, err := ds.StoreDistributed(
        context.Background(),
        "0xtest_user",
        1,
        testData,
    )

    assert.NoError(t, err)
    assert.NotNil(t, chunk.ContractID)

    // 6. Verify storage contract on blockchain
    contract, err := blockchain.GetStorageContract(chunk.ContractID)
    assert.NoError(t, err)
    assert.Equal(t, chunk.MerkleRoot, contract.MerkleRoot)

    // 7. Issue challenge
    challengeID, _ := blockchain.IssueChallenge(
        contract.ContractID,
        contract.AssignedNodes[0],
        0, // First shard
    )

    // 8. Wait for challenge response (automated by node)
    time.Sleep(10 * time.Second)

    // 9. Verify challenge resolved successfully
    challenge, _ := blockchain.GetChallenge(challengeID)
    assert.True(t, challenge.IsResolved)
    assert.True(t, challenge.WasSuccessful)

    // 10. Retrieve data
    retrieved, err := ds.RetrieveDistributed(
        context.Background(),
        "0xtest_user",
        1,
    )

    assert.NoError(t, err)
    assert.Equal(t, testData, retrieved)

    // 11. Verify rewards distributed
    nodeInfo, _ := blockchain.GetNodeInfo(nodes[0].Address())
    assert.Greater(t, nodeInfo.TotalEarnedRewards.Int64(), int64(0))
}
```

### Smart Contract Tests (Hardhat)

```typescript
// test/StorageRewards.test.ts

describe("StorageRewards", () => {
  let storageRewards: StorageRewards;
  let owner: SignerWithAddress;
  let node1: SignerWithAddress;

  beforeEach(async () => {
    [owner, node1] = await ethers.getSigners();

    const StorageRewards = await ethers.getContractFactory("StorageRewards");
    storageRewards = await StorageRewards.deploy(
      rewardToken.address,
      ethers.utils.parseEther("1000") // 1000 tokens/day
    );
  });

  it("should register a node with stake", async () => {
    const nodeID = ethers.utils.keccak256(ethers.utils.toUtf8Bytes("node1"));
    const stake = ethers.utils.parseEther("1000");
    const capacity = 100; // 100 GB

    await expect(
      storageRewards.connect(node1).registerNode(nodeID, capacity, {
        value: stake,
      })
    )
      .to.emit(storageRewards, "NodeRegistered")
      .withArgs(node1.address, nodeID, stake, capacity);

    const nodeInfo = await storageRewards.nodes(node1.address);
    expect(nodeInfo.stakedAmount).to.equal(stake);
    expect(nodeInfo.storageCapacityGB).to.equal(capacity);
    expect(nodeInfo.isActive).to.be.true;
  });

  it("should create storage contract", async () => {
    // Register nodes first
    // ...

    const merkleRoot = ethers.utils.keccak256(
      ethers.utils.toUtf8Bytes("test_data")
    );
    const dataSize = 1024 * 1024; // 1 MB
    const duration = 30; // 30 days
    const assignedNodes = [nodeID1, nodeID2, /* ... 15 nodes */];

    const cost = await storageRewards.calculateStorageCost(dataSize, duration);

    await expect(
      storageRewards.createStorageContract(
        merkleRoot,
        dataSize,
        duration,
        assignedNodes,
        { value: cost }
      )
    ).to.emit(storageRewards, "StorageContractCreated");
  });

  it("should slash node for failed challenge", async () => {
    // Setup: Register node, create contract, issue challenge
    // ...

    const initialStake = await storageRewards.nodes(node1.address).stakedAmount;

    // Node fails to respond to challenge (timeout)
    await time.increase(6 * 60); // 6 minutes

    await challengeManager.expireChallenge(challengeID);

    const finalStake = await storageRewards.nodes(node1.address).stakedAmount;
    const slashedAmount = initialStake.mul(10).div(100); // 10%

    expect(finalStake).to.equal(initialStake.sub(slashedAmount));
  });
});
```

---

## Deployment Plan

### Phase 3A: Testnet Deployment (Week 1-2)

**Objectives:**
- Deploy contracts to testnet (Sepolia/Goerli)
- Test with 5-10 nodes
- Verify challenge system works
- Identify bugs and issues

**Steps:**

```bash
# 1. Deploy contracts to testnet
cd contracts
npx hardhat deploy --network sepolia

# Outputs:
# StorageRewards: 0x1234...
# ProofVerifier: 0x5678...
# ChallengeManager: 0x9abc...

# 2. Update configuration
cat > config/testnet.json <<EOF
{
  "network": "sepolia",
  "rpcURL": "https://sepolia.infura.io/v3/YOUR_KEY",
  "contracts": {
    "storageRewards": "0x1234...",
    "proofVerifier": "0x5678...",
    "challengeManager": "0x9abc..."
  },
  "challengeInterval": "24h",
  "rewardPool": "1000"
}
EOF

# 3. Start test nodes
./storage-node --port 9001 --key node1.key --stake 1000 --config config/testnet.json &
./storage-node --port 9002 --key node2.key --stake 1000 --config config/testnet.json &
./storage-node --port 9003 --key node3.key --stake 1000 --config config/testnet.json &

# 4. Run integration tests
go test ./... -tags=integration -v

# 5. Monitor challenges
node scripts/monitor-challenges.js --network sepolia
```

**Success Criteria:**
- [ ] All contracts deployed successfully
- [ ] 10+ nodes registered and staking
- [ ] 100+ challenges issued and resolved
- [ ] 95%+ challenge success rate
- [ ] No critical bugs found
- [ ] Gas costs < $0.01 per challenge

### Phase 3B: Mainnet Preparation (Week 3)

**Security Audit:**

```
Audit Checklist:
├─ Smart Contract Audit (by CertiK/OpenZeppelin)
│  ├─ Reentrancy vulnerabilities
│  ├─ Integer overflow/underflow
│  ├─ Access control issues
│  ├─ Front-running risks
│  └─ Gas optimization
│
├─ Network Security Review
│  ├─ DDoS protection
│  ├─ Sybil resistance
│  ├─ Eclipse attack prevention
│  └─ Rate limiting
│
└─ Economic Model Validation
   ├─ Reward distribution fairness
   ├─ Slashing mechanism effectiveness
   ├─ Token emission schedule
   └─ Long-term sustainability
```

**Documentation:**

1. **Node Operator Guide**
   - Hardware requirements
   - Setup instructions
   - Staking process
   - Earnings calculation
   - Troubleshooting

2. **User Guide**
   - How to store data
   - Pricing calculator
   - Data retrieval
   - Understanding storage contracts

3. **Developer Documentation**
   - API reference
   - SDK usage
   - Smart contract ABI
   - Integration examples

### Phase 3C: Mainnet Launch (Week 4)

**Launch Sequence:**

```
Day 1: Contract Deployment
├─ Deploy to mainnet
├─ Verify on Etherscan
├─ Initialize with 1M ZENT reward pool
└─ Set parameters (challenge frequency, etc.)

Day 2-3: Bootstrap Phase
├─ Team runs 5 bootstrap nodes
├─ Community whitelisted to join (KYC'd operators)
├─ Max 20 nodes initially
└─ Monitor for 48 hours

Day 4-7: Public Launch
├─ Open registration to all
├─ Marketing campaign
├─ Monitor growth
└─ Daily standups for quick issue resolution

Day 8-30: Stabilization
├─ Monitor network health
├─ Adjust parameters if needed
├─ Collect feedback
└─ Plan Phase 4 improvements
```

**Monitoring Dashboard:**

```typescript
// dashboard/metrics.ts

export interface NetworkMetrics {
  // Node metrics
  totalNodes: number;
  activeNodes: number;
  totalStaked: BigNumber;
  totalStorageGB: number;

  // Challenge metrics
  totalChallenges: number;
  successfulChallenges: number;
  failedChallenges: number;
  averageResponseTime: number;

  // Economic metrics
  dailyRewardPool: BigNumber;
  totalRewardsDistributed: BigNumber;
  averageNodeEarnings: BigNumber;

  // Storage metrics
  activeContracts: number;
  totalDataStored: number;
  averageContractDuration: number;
}

async function getNetworkMetrics(): Promise<NetworkMetrics> {
  const contract = getStorageRewardsContract();

  return {
    totalNodes: await contract.getTotalNodes(),
    activeNodes: await contract.getActiveNodes(),
    totalStaked: await contract.getTotalStaked(),
    // ... fetch all metrics
  };
}
```

---

## Performance Requirements

### Latency Targets

| Operation | Target | P95 | Max |
|-----------|--------|-----|-----|
| Challenge issuance | 1s | 2s | 5s |
| Proof generation | 30s | 60s | 120s |
| Proof verification | 5s | 10s | 30s |
| Challenge response | 4min | 4.5min | 5min |
| Reward calculation | 10s | 30s | 60s |
| Contract read | 100ms | 500ms | 1s |
| Contract write | 15s | 30s | 60s |

### Throughput Targets

| Metric | Target | Peak |
|--------|--------|------|
| Challenges/day | 10,000 | 50,000 |
| Proofs verified/sec | 100 | 500 |
| Nodes supported | 1,000 | 10,000 |
| Storage contracts/day | 1,000 | 10,000 |
| Data stored/day | 1 TB | 10 TB |

### Gas Cost Targets

| Operation | Target (gas) | Cost @ 20 gwei | Cost @ 100 gwei |
|-----------|--------------|----------------|-----------------|
| Register node | 200,000 | $0.004 | $0.02 |
| Create contract | 150,000 | $0.003 | $0.015 |
| Issue challenge | 50,000 | $0.001 | $0.005 |
| Respond challenge | 100,000 | $0.002 | $0.01 |
| Verify proof | 30,000 | $0.0006 | $0.003 |
| Claim rewards | 80,000 | $0.0016 | $0.008 |

### Scalability Projections

```
Year 1:
- Nodes: 100 → 1,000
- Storage: 10 TB → 100 TB
- Challenges/day: 1,500 → 15,000
- Gas costs/day: $50 → $500

Year 3:
- Nodes: 10,000
- Storage: 1 PB
- Challenges/day: 150,000
- Gas costs/day: $5,000 (may need L2 scaling)

L2 Migration Strategy:
- Move to Optimism/Arbitrum if mainnet costs > $10K/day
- Expected gas savings: 10-100x
- Timeline: Year 2-3
```

---

## Next Steps

### Implementation Roadmap (4 weeks)

**Week 1: Smart Contracts**
- Day 1-2: Write StorageRewards.sol
- Day 3: Write ProofVerifier.sol
- Day 4: Write ChallengeManager.sol
- Day 5: Unit tests for all contracts
- Day 6-7: Integration tests, gas optimization

**Week 2: Go Integration**
- Day 1-2: Merkle tree implementation
- Day 3: Blockchain client
- Day 4: Challenge handler
- Day 5: Update distributed storage
- Day 6-7: Integration testing

**Week 3: Testing & Security**
- Day 1-3: Comprehensive testing
- Day 4-5: Security review
- Day 6-7: Testnet deployment

**Week 4: Launch**
- Day 1-2: Mainnet deployment
- Day 3-7: Monitoring & support

### Decision Points

Before starting implementation, confirm:

1. **Blockchain Choice**
   - [ ] Ethereum mainnet
   - [ ] Polygon
   - [ ] Optimism/Arbitrum
   - [ ] Other L2

2. **Token Model**
   - [ ] New ERC-20 token
   - [ ] Use existing token
   - [ ] Multi-token support

3. **Governance**
   - [ ] DAO for parameter updates
   - [ ] Multisig for admin functions
   - [ ] Timelock for security

4. **Launch Strategy**
   - [ ] Gradual rollout (whitelist first)
   - [ ] Public launch
   - [ ] Incentivized testnet first

---

## Conclusion

This technical specification provides a complete blueprint for implementing Phase 3 blockchain integration. The system is designed to be:

✅ **Secure** - Cryptographic proofs, staking, slashing
✅ **Scalable** - L2-ready, efficient gas usage
✅ **Economically Sustainable** - Token rewards, fee model
✅ **Production-Ready** - Comprehensive testing, monitoring

**Ready to build?** Let's start with Week 1 smart contract development!
