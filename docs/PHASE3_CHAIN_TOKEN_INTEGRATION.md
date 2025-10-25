# Phase 3: CHAIN Token Integration Guide

**Version**: 1.0
**Last Updated**: 2025-01-20
**Existing Token**: CHAIN
**Contract**: `0xc91f1effddc4a727f4de78e400137390779ec062`
**Network**: Ethereum Mainnet

---

## Overview

Phase 3 will integrate with your **existing CHAIN token** for storage network rewards instead of deploying a new token. This document outlines the adjusted economics and integration approach.

---

## CHAIN Token Specifications

```
Token Name: CHAIN
Symbol: CHAIN
Contract Address: 0xc91f1effddc4a727f4de78e400137390779ec062
Network: Ethereum Mainnet
Decimals: 12 (note: most ERC-20 use 18)
Max Total Supply: 5,000,000 CHAIN
```

### Key Differences from Standard ERC-20

```solidity
// Standard ERC-20 uses 18 decimals
1 token = 1 * 10^18 wei (standard)

// CHAIN token uses 12 decimals
1 CHAIN = 1 * 10^12 units

// Conversion examples:
1000 CHAIN = 1000 * 10^12 units = 1,000,000,000,000,000
0.5 CHAIN = 0.5 * 10^12 units = 500,000,000,000
```

---

## Adjusted Token Economics

### Original Plan vs CHAIN Token Plan

| Parameter | Original (ZENT) | Adjusted (CHAIN) | Change |
|-----------|-----------------|------------------|--------|
| Total Supply | 1,000,000,000 | 5,000,000 | 200x smaller |
| Decimals | 18 | 12 | Different precision |
| Storage Reward Pool | 400M (40%) | 2,000,000 (40%) | Proportional |
| Team/Dev | 250M (25%) | 1,250,000 (25%) | Proportional |
| Community | 200M (20%) | 1,000,000 (20%) | Proportional |
| Liquidity | 100M (10%) | 500,000 (10%) | Proportional |
| Reserve | 50M (5%) | 250,000 (5%) | Proportional |

### Updated Distribution Plan

```
Total CHAIN Supply: 5,000,000

Allocation:
├─ 2,000,000 (40%) - Storage Node Rewards (vested 10 years)
├─ 1,250,000 (25%) - Team & Development (4-year vesting)
├─ 1,000,000 (20%) - Community & Ecosystem
├─   500,000 (10%) - Liquidity Pools
└─   250,000 (5%)  - Reserve Fund

Storage Reward Emission Schedule:
Year 1:  547.95 CHAIN/day  (200,000 / 365)
Year 2:  410.96 CHAIN/day  (150,000 / 365) [-25%]
Year 3:  308.22 CHAIN/day  (112,500 / 365) [-25%]
Year 4:  231.16 CHAIN/day  ( 84,375 / 365) [-25%]
...
Year 10:  27.40 CHAIN/day  ( 10,000 / 365)
```
no 
---

## Updated Reward Calculations

### Daily Reward Formula (Same Structure)

```
Node Daily Reward = Base + Performance + Challenge + Longevity

Where each component uses CHAIN instead of ZENT
```

### Example: Year 1 Rewards

**Network State:**
- Total network storage: 10,000 GB
- Total nodes: 100
- Daily reward pool: 547.95 CHAIN

**Node A (100 GB, 99.5% uptime, 100% challenges, 30 days old):**

```
Base Reward = (100 / 10000) × 547.95 × 0.40
            = 0.01 × 547.95 × 0.40
            = 2.19 CHAIN

Performance = 0.995 × 547.95 × 0.30
            = 163.57 CHAIN

Challenge = 1.0 × 547.95 × 0.20
          = 109.59 CHAIN

Longevity = (30/365) × 547.95 × 0.10
          = 4.51 CHAIN

Total Daily = 2.19 + 163.57 + 109.59 + 4.51
            = 279.86 CHAIN/day

Monthly (30 days) = 279.86 × 30
                  = 8,395.8 CHAIN/month
```

**Value Analysis:**

If CHAIN trades at different price points:

| CHAIN Price | Monthly Earnings | Annual Earnings |
|-------------|------------------|-----------------|
| $0.10 | $839.58 | $10,075 |
| $1.00 | $8,395.80 | $100,750 |
| $10.00 | $83,958.00 | $1,007,500 |
| $50.00 | $419,790.00 | $5,037,500 |

**Note:** With smaller total supply (5M vs 1B), CHAIN should have higher unit price potential.

---

## Smart Contract Integration

### StorageRewards Contract Updates

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract StorageRewards is ReentrancyGuard {

    // Use existing CHAIN token
    IERC20 public constant CHAIN_TOKEN = IERC20(
        0xc91f1effddc4a727f4de78e400137390779ec062
    );

    // Adjust for 12 decimals instead of 18
    uint256 public constant DECIMALS = 12;
    uint256 public constant ONE_CHAIN = 10**12; // 1 CHAIN token

    // Minimum stake adjusted for CHAIN economics
    uint256 public constant MINIMUM_STAKE = 100 * ONE_CHAIN; // 100 CHAIN

    // Daily reward pool (adjustable by governance)
    uint256 public dailyRewardPool = 548 * ONE_CHAIN; // ~548 CHAIN/day (Year 1)

    // ... rest of contract

    /**
     * @notice Register node with CHAIN tokens
     * @dev Node must approve this contract to spend CHAIN first
     */
    function registerNode(
        bytes32 nodeID,
        uint256 storageCapacityGB
    ) external {
        // Transfer CHAIN tokens from node operator
        require(
            CHAIN_TOKEN.transferFrom(msg.sender, address(this), MINIMUM_STAKE),
            "CHAIN transfer failed"
        );

        nodes[msg.sender] = Node({
            owner: msg.sender,
            nodeID: nodeID,
            stakedAmount: MINIMUM_STAKE,
            storageCapacityGB: storageCapacityGB,
            successfulChallenges: 0,
            failedChallenges: 0,
            totalEarnedRewards: 0,
            registeredAt: block.timestamp,
            unstakeRequestedAt: 0,
            isActive: true,
            reputationScore: 10000
        });

        nodeIDToAddress[nodeID] = msg.sender;
        totalNetworkStorageGB += storageCapacityGB;

        emit NodeRegistered(msg.sender, nodeID, MINIMUM_STAKE, storageCapacityGB);
    }

    /**
     * @notice Claim accumulated CHAIN rewards
     */
    function claimRewards() external nonReentrant {
        Node storage node = nodes[msg.sender];
        require(node.owner != address(0), "Not registered");

        uint256 pendingRewards = calculatePendingRewards(msg.sender);
        require(pendingRewards > 0, "No rewards to claim");

        // Transfer CHAIN tokens to node operator
        require(
            CHAIN_TOKEN.transfer(msg.sender, pendingRewards),
            "Reward transfer failed"
        );

        node.totalEarnedRewards += pendingRewards;

        emit RewardsClaimed(msg.sender, pendingRewards, block.timestamp);
    }

    /**
     * @notice Slash node stake (failed challenge)
     */
    function slashNode(address nodeAddress, uint256 slashAmount) internal {
        Node storage node = nodes[nodeAddress];

        // Reduce stake
        node.stakedAmount -= slashAmount;

        // Slashed tokens go to treasury for redistribution
        require(
            CHAIN_TOKEN.transfer(treasury, slashAmount),
            "Slash transfer failed"
        );

        emit NodeSlashed(nodeAddress, node.nodeID, slashAmount, "Failed challenge");
    }
}
```

### Key Changes for CHAIN Integration

1. **No New Token Deployment** - Use existing CHAIN contract
2. **12 Decimal Precision** - All calculations use `10^12` instead of `10^18`
3. **Lower Stake Requirements** - 100 CHAIN minimum (vs 1,000 ZENT)
4. **Smaller Daily Pools** - ~548 CHAIN/day (vs 109,589 ZENT/day)
5. **ERC-20 Transfers** - Use `transferFrom()` and `transfer()` for CHAIN

---

## Go Client Integration

### Updated Blockchain Client

```go
// pkg/blockchain/chain_token.go

package blockchain

import (
    "math/big"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/accounts/abi/bind"
)

var (
    // CHAIN token contract address (Ethereum Mainnet)
    ChainTokenAddress = common.HexToAddress("0xc91f1effddc4a727f4de78e400137390779ec062")

    // CHAIN uses 12 decimals (not 18!)
    ChainDecimals = int64(12)

    // 1 CHAIN = 10^12 units
    OneChain = new(big.Int).Exp(big.NewInt(10), big.NewInt(ChainDecimals), nil)
)

// ChainAmount converts human-readable CHAIN to contract units
// Example: ChainAmount(100.5) = 100.5 * 10^12
func ChainAmount(chain float64) *big.Int {
    // Convert to smallest unit
    units := chain * float64(OneChain.Int64())
    return big.NewInt(int64(units))
}

// ChainToHuman converts contract units to human-readable CHAIN
// Example: ChainToHuman(100500000000000) = 100.5 CHAIN
func ChainToHuman(units *big.Int) float64 {
    return float64(units.Int64()) / float64(OneChain.Int64())
}

// ApproveChainForStaking approves StorageRewards contract to spend CHAIN
func (c *Client) ApproveChainForStaking(
    ctx context.Context,
    amount *big.Int,
    privateKey *ecdsa.PrivateKey,
) (*types.Transaction, error) {
    auth, err := bind.NewKeyedTransactorWithChainID(privateKey, c.chainID)
    if err != nil {
        return nil, err
    }

    // Get CHAIN token contract
    chainToken, err := NewERC20(ChainTokenAddress, c.eth)
    if err != nil {
        return nil, err
    }

    // Approve StorageRewards contract to spend CHAIN
    return chainToken.Approve(auth, c.storageRewards.Address(), amount)
}

// RegisterNodeWithChain registers node using CHAIN tokens
func (c *Client) RegisterNodeWithChain(
    ctx context.Context,
    nodeID [32]byte,
    storageCapacityGB *big.Int,
    stakeAmountChain *big.Int,
    privateKey *ecdsa.PrivateKey,
) (*types.Transaction, error) {
    // First, approve CHAIN spending
    approveTx, err := c.ApproveChainForStaking(ctx, stakeAmountChain, privateKey)
    if err != nil {
        return nil, fmt.Errorf("approval failed: %w", err)
    }

    // Wait for approval confirmation
    _, err = c.WaitForReceipt(ctx, approveTx.Hash())
    if err != nil {
        return nil, fmt.Errorf("approval tx failed: %w", err)
    }

    // Now register node
    auth, err := bind.NewKeyedTransactorWithChainID(privateKey, c.chainID)
    if err != nil {
        return nil, err
    }

    return c.storageRewards.RegisterNode(auth, nodeID, storageCapacityGB)
}
```

### Updated Storage Node Main

```go
// cmd/storage-node/main.go

func main() {
    // ... parse flags ...

    stakeCHAIN := flag.Float64("stake", 100.0, "CHAIN tokens to stake")

    // ... initialize blockchain client ...

    // Convert CHAIN to contract units (12 decimals)
    stakeAmount := blockchain.ChainAmount(*stakeCHAIN)

    fmt.Printf("Registering node with %f CHAIN stake...\n", *stakeCHAIN)
    fmt.Printf("Contract units: %s\n", stakeAmount.String())

    // Register with CHAIN tokens
    tx, err := bc.RegisterNodeWithChain(
        context.Background(),
        nodeIDBytes,
        big.NewInt(int64(*storageGB)),
        stakeAmount,
        privateKey,
    )

    if err != nil {
        log.Fatalf("Failed to register: %v", err)
    }

    fmt.Printf("✅ Registered with %f CHAIN staked\n", *stakeCHAIN)

    // ... rest of node setup ...
}
```

---

## Updated Staking Requirements

### Minimum Stake Tiers

Given the smaller total supply, we adjust stake requirements:

```
Tier 1 (Small Node):
- Stake: 100 CHAIN
- Max Storage: 50 GB
- Expected Monthly: ~420 CHAIN ($420-$42K depending on price)

Tier 2 (Medium Node):
- Stake: 500 CHAIN
- Max Storage: 250 GB
- Expected Monthly: ~2,100 CHAIN ($2.1K-$210K)

Tier 3 (Large Node):
- Stake: 2,000 CHAIN
- Max Storage: 1 TB
- Expected Monthly: ~8,400 CHAIN ($8.4K-$840K)

Tier 4 (Enterprise Node):
- Stake: 10,000 CHAIN
- Max Storage: 10 TB
- Expected Monthly: ~42,000 CHAIN ($42K-$4.2M)
```

### Slashing Adjusted for CHAIN

```
Failed Challenge: -10% of stake
- Tier 1: -10 CHAIN
- Tier 2: -50 CHAIN
- Tier 3: -200 CHAIN
- Tier 4: -1,000 CHAIN

Extended Downtime (>24h): -1% of stake per hour
- Tier 1: -1 CHAIN/hour
- Tier 2: -5 CHAIN/hour
- Tier 3: -20 CHAIN/hour
- Tier 4: -100 CHAIN/hour
```

---

## Treasury Management

### CHAIN Token Treasury Contract

```solidity
// contracts/ChainTreasury.sol

contract ChainTreasury {
    IERC20 public constant CHAIN = IERC20(
        0xc91f1effddc4a727f4de78e400137390779ec062
    );

    // Revenue sources
    uint256 public totalUserPayments;      // From storage contracts
    uint256 public totalSlashedTokens;     // From penalties
    uint256 public totalDonations;         // Community contributions

    // Reward pool management
    uint256 public dailyRewardBudget;
    uint256 public lastDistribution;

    /**
     * @notice Load treasury with CHAIN for rewards
     * @dev Team/DAO transfers CHAIN to this contract
     */
    function loadRewardPool(uint256 amount) external {
        require(
            CHAIN.transferFrom(msg.sender, address(this), amount),
            "Transfer failed"
        );

        emit RewardPoolLoaded(amount, block.timestamp);
    }

    /**
     * @notice Distribute daily rewards to StorageRewards contract
     */
    function distributeDailyRewards() external {
        require(
            block.timestamp >= lastDistribution + 1 days,
            "Too early"
        );

        require(
            CHAIN.transfer(storageRewardsContract, dailyRewardBudget),
            "Distribution failed"
        );

        lastDistribution = block.timestamp;

        emit DailyRewardsDistributed(dailyRewardBudget, block.timestamp);
    }

    /**
     * @notice Receive user payments for storage
     */
    function receiveStoragePayment(uint256 amount) external {
        require(
            CHAIN.transferFrom(msg.sender, address(this), amount),
            "Payment failed"
        );

        totalUserPayments += amount;

        emit StoragePaymentReceived(msg.sender, amount);
    }

    /**
     * @notice Receive slashed tokens from penalties
     */
    function receiveSlashedTokens(uint256 amount) external {
        require(msg.sender == storageRewardsContract, "Not authorized");

        totalSlashedTokens += amount;

        // Redistribute to reward pool
        dailyRewardBudget += amount / 365; // Spread over a year
    }
}
```

---

## User Payment Pricing

### Storage Cost in CHAIN

```
Formula:
Cost = (Size GB × Duration Days × 0.02 CHAIN) / 30

Examples:

1 GB for 30 days:
= (1 × 30 × 0.02) / 30
= 0.02 CHAIN

10 GB for 90 days:
= (10 × 90 × 0.02) / 30
= 0.6 CHAIN

100 GB for 365 days:
= (100 × 365 × 0.02) / 30
= 24.33 CHAIN

1 TB for 365 days:
= (1000 × 365 × 0.02) / 30
= 243.33 CHAIN
```

### Fiat Equivalent (at different CHAIN prices)

| Storage | Duration | CHAIN Cost | @ $1 | @ $10 | @ $50 |
|---------|----------|------------|------|-------|-------|
| 1 GB | 30 days | 0.02 | $0.02 | $0.20 | $1.00 |
| 10 GB | 30 days | 0.20 | $0.20 | $2.00 | $10.00 |
| 100 GB | 30 days | 2.00 | $2.00 | $20.00 | $100.00 |
| 1 TB | 30 days | 20.00 | $20.00 | $200.00 | $1,000.00 |
| 1 TB | 365 days | 243.33 | $243.33 | $2,433.30 | $12,166.50 |

---

## Migration from Original Spec

### What Changes

❌ **Remove:**
- New token deployment
- Token contract creation
- 18 decimal handling
- 1B token supply calculations

✅ **Keep:**
- All smart contract logic (just use CHAIN)
- Merkle proof system (unchanged)
- Challenge-response protocol (unchanged)
- Network architecture (unchanged)

✅ **Update:**
- Decimal precision (18 → 12)
- Token amounts (divided by 200)
- Minimum stakes (1000 → 100)
- Daily rewards (109,589 → 548)

### Code Changes Required

```diff
// contracts/StorageRewards.sol

- IERC20 public rewardToken;
+ IERC20 public constant CHAIN_TOKEN = IERC20(0xc91f1effddc4a727f4de78e400137390779ec062);

- uint256 public constant MINIMUM_STAKE = 1000 * 10**18;
+ uint256 public constant MINIMUM_STAKE = 100 * 10**12;

- uint256 public constant ONE_TOKEN = 10**18;
+ uint256 public constant ONE_CHAIN = 10**12;

- uint256 public dailyRewardPool = 109589 * ONE_TOKEN;
+ uint256 public dailyRewardPool = 548 * ONE_CHAIN;
```

---

## Initial Treasury Setup

### Team Needs to Allocate CHAIN

To start Phase 3, the team/DAO needs to:

1. **Allocate Storage Rewards Pool**
   ```
   Year 1 budget: 200,000 CHAIN

   Options:
   a) Load full year upfront: 200,000 CHAIN to treasury
   b) Load quarterly: 50,000 CHAIN every 3 months
   c) Load monthly: ~16,667 CHAIN every month
   ```

2. **Approve Treasury Contract**
   ```solidity
   // Team multisig approves treasury to spend CHAIN
   CHAIN.approve(treasuryAddress, 200000 * 10**12);
   ```

3. **Set Daily Reward Rate**
   ```solidity
   // Set initial daily distribution
   treasury.setDailyRewardBudget(548 * 10**12); // 548 CHAIN/day
   ```

### Governance Control

```solidity
// contracts/ChainGovernance.sol

contract ChainGovernance {
    // Multisig or DAO can adjust parameters

    function updateDailyRewardPool(uint256 newAmount) external onlyGovernance {
        require(newAmount <= maxDailyReward, "Too high");
        treasury.setDailyRewardBudget(newAmount);
    }

    function updateMinimumStake(uint256 newMinimum) external onlyGovernance {
        require(newMinimum >= 50 * ONE_CHAIN, "Too low");
        require(newMinimum <= 1000 * ONE_CHAIN, "Too high");
        storageRewards.setMinimumStake(newMinimum);
    }
}
```

---

## Deployment Checklist

### Pre-Deployment

- [ ] Verify CHAIN token is accessible on mainnet
- [ ] Confirm team controls enough CHAIN for Year 1 rewards (200K)
- [ ] Set up multisig for treasury management
- [ ] Prepare CHAIN token approvals

### Smart Contract Deployment

- [ ] Deploy StorageRewards (references CHAIN at 0xc91f...)
- [ ] Deploy ProofVerifier
- [ ] Deploy ChallengeManager
- [ ] Deploy ChainTreasury
- [ ] Deploy ChainGovernance
- [ ] Connect all contracts

### Treasury Initialization

- [ ] Transfer 200,000 CHAIN to treasury (Year 1 budget)
- [ ] Set dailyRewardBudget = 548 CHAIN
- [ ] Verify treasury can distribute to StorageRewards
- [ ] Test reward claims work correctly

### Testing

- [ ] Register test node with 100 CHAIN stake
- [ ] Create test storage contract
- [ ] Issue test challenge
- [ ] Verify rewards distributed in CHAIN
- [ ] Test slashing returns CHAIN to treasury

---

## Risk Analysis with CHAIN Token

### Risks

1. **Limited Supply (5M vs 1B)**
   - ✅ Pro: Higher potential token value
   - ⚠️ Con: Rewards pool depletes faster if network grows rapidly
   - Mitigation: Adjust daily emission based on network size

2. **12 Decimals (vs standard 18)**
   - ⚠️ Con: Less precision for micro-transactions
   - ✅ Pro: Simpler human-readable amounts
   - Mitigation: Ensure all contracts handle 12 decimals correctly

3. **Existing Token Liquidity**
   - ⚠️ Con: Need to ensure sufficient liquidity for nodes to exit
   - ✅ Pro: Already has market value and holders
   - Mitigation: Work with DEXs to provide liquidity

4. **Governance**
   - ⚠️ Con: Token holders may not align with storage network goals
   - ✅ Pro: Existing community can participate
   - Mitigation: Clear governance structure for network parameters

### Opportunities

1. **Token Utility Expansion**
   - Storage rewards create real use case for CHAIN
   - Drives demand (nodes need to buy CHAIN to stake)
   - Users need CHAIN to pay for storage

2. **Value Accrual**
   - Slashed tokens reduce circulating supply
   - Storage payments increase treasury holdings
   - Network revenue can buy back CHAIN

3. **Community Alignment**
   - Existing CHAIN holders can become storage nodes
   - Storage network success increases token value
   - Dual value proposition: token appreciation + staking rewards

---

## Next Steps

### Immediate Actions

1. **Confirm CHAIN Access**
   - [ ] Verify team wallet has 200K+ CHAIN for rewards
   - [ ] Check CHAIN token approval capabilities
   - [ ] Confirm no restrictions on transfers/staking

2. **Update Smart Contracts**
   - [ ] Replace all token references with CHAIN address
   - [ ] Update decimals from 18 to 12
   - [ ] Adjust all reward calculations
   - [ ] Test with CHAIN token on testnet first

3. **Economic Model Review**
   - [ ] Validate 548 CHAIN/day is sustainable
   - [ ] Model growth scenarios (100 → 1000 nodes)
   - [ ] Plan for reward pool refills

4. **Community Communication**
   - [ ] Announce storage network to CHAIN holders
   - [ ] Explain staking opportunity
   - [ ] Create node operator incentive program

---

## Summary

✅ **Phase 3 can proceed with CHAIN token**
✅ **All technical architecture remains the same**
✅ **Economics adjusted for 5M supply**
✅ **No new token deployment needed**

### Key Numbers

- **Minimum Stake**: 100 CHAIN (vs 1000 ZENT)
- **Daily Rewards**: ~548 CHAIN (vs 109,589 ZENT)
- **Year 1 Budget**: 200,000 CHAIN from existing supply
- **Storage Pricing**: 0.02 CHAIN per GB per 30 days

### Ready to Build

The technical specification in `PHASE3_TECHNICAL_SPEC.md` is still valid - just replace:
- ZENT → CHAIN
- 10^18 → 10^12
- Contract deployment → Use existing 0xc91f...
- Amounts divided by ~200

**Shall we update the main spec document and start implementing?**
