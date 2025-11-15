# Obscura Protocol: Whitepaper & Decentralization Roadmap

## Abstract

This document summarizes the Obscura whitepaper and details the **phased transition from centralized MVP to fully decentralized protocol**. Obscura synthesizes BFT consensus, TEE security, and ZK proofs to create a verifiable reputation layer for DeFi.

---

## Core Protocol Components

### 1. Sentinel DON (Decentralized Oracle Network)
**Purpose**: Replace centralized trade data ingestion with BFT consensus

**Mechanism**: Chainlink OCR-style consensus
- Leader election → Distributed observation → Signed reports → Aggregated proof
- **BFT Tolerance**: n = 3f + 1 (tolerates f Byzantine nodes)
- **Cryptoeconomic Security**: Node operators stake ZEN tokens

**Slashing Formula**:
```
δi(t) = |vi - V̂| / V̂  (deviation score)

If Σ δi(t) > Θ → Slash node i's stake
   t∈T
```

### 2. Citadel (TEE Confidential Compute)
**Purpose**: Hardware-isolated credential management

**Mechanism**: AWS Nitro Enclaves
- Hardware-enforced memory encryption
- Cryptographic attestation (PCR verification)
- Credentials never leave enclave boundary

**Attestation**: Only code with matching PCR hashes can access KMS encryption keys

### 3. Alchemist (ZK Proof System)
**Purpose**: Private, verifiable performance proofs

**Mechanism**: Noir circuits + Recursive aggregation
- Individual trade proofs aggregated into single O(1) verification
- BN254 curve (~128-bit security)
- On-chain verification on Horizen L3

---

## Decentralization Roadmap

### Phase 1: Centralized MVP (Q4 2025) ✅ **CURRENT**

**Status**: Deployed and operational

**Architecture**:
```
Centralized Components:
├─ Sentinel: Single instance (Obscura-operated)
├─ Citadel: Single TEE (AWS Nitro Enclave)
├─ Conductor: Single orchestrator
└─ Alchemist: Single ZK prover

Decentralized Components:
├─ Smart Contracts: Deployed on Horizen L3
└─ User Reputation: On-chain, immutable
```

**Characteristics**:
- ✅ Fast iteration and feature development
- ✅ Lower operational overhead
- ✅ Proven security (TEE + ZK)
- ⚠️ Single point of failure (Sentinel)
- ⚠️ Trust in Obscura entity for data integrity

**Why Start Centralized**:
- Validate product-market fit
- Refine protocols before decentralization
- Build user base and reputation dataset
- Demonstrate TEE + ZK integration works

---

### Phase 2: Sentinel DON Decentralization (Q2 2026)

**Status**: 🔄 In Development

**Objective**: Replace centralized Sentinel with permissionless BFT oracle network

#### Components to Decentralize

**2.1 Node Operator Network**

**Requirements for Node Operators**:
```yaml
Hardware:
  CPU: 8+ cores
  RAM: 32GB+
  Storage: 1TB SSD
  Network: 100 Mbps+ (low latency)

Software:
  Sentinel Node: Docker image (Obscura-provided)
  Monitoring: Prometheus + Grafana
  API Keys: Exchange APIs for data access

Economic:
  Stake: 10,000 ZEN minimum (~$100,000)
  Collateral: Locked in StakingContract
  Rewards: % of protocol fees + inflation
```

**Node Operator Onboarding Flow**:
```
1. Stake ZEN Tokens
   └─▶ Lock 10,000 ZEN in StakingContract

2. Deploy Node Software
   ├─▶ Docker: docker pull obscura/sentinel-node:v2.0
   ├─▶ Configure: Set exchange API keys (read-only)
   └─▶ Start: docker-compose up -d

3. Register Node On-Chain
   └─▶ Call: NodeRegistry.register(nodeAddress, metadata)

4. Attestation & Health Check
   ├─▶ Node submits first observation
   ├─▶ Network validates
   └─▶ If valid: Node becomes active in rotation

5. Earn Rewards
   ├─▶ Participate in consensus rounds
   ├─▶ Sign accurate observations
   └─▶ Earn fees + inflation rewards
```

**2.2 BFT Consensus Implementation**

**OCR (Off-Chain Reporting) Protocol**:
```
Round N (every 60 seconds):

1. Leader Selection
   • Deterministic: leader = hash(epoch + round) mod n
   • Rotates each round for fairness

2. Observation Phase (0-20s)
   • Leader broadcasts: "Request observations for user X"
   • Followers fetch trade data independently
   • Each node signs observation: sig(trade_data, private_key)

3. Report Phase (20-40s)
   • Followers send signed observations to leader
   • Leader waits for quorum: 2f+1 signatures
   • Leader computes median/consensus value
   • Leader creates report: Report(median, [signatures])

4. Attestation Phase (40-55s)
   • Leader broadcasts report to followers
   • Followers validate: check signatures + consensus logic
   • If valid: Followers sign report
   • Leader collects 2f+1 signatures on report

5. Submission Phase (55-60s)
   • Single transmitter submits to Horizen L3
   • Gas cost: ~280k gas (amortized across all trades)
   • On-chain contract verifies quorum signatures
```

**2.3 Slashing & Rewards**

**Reward Distribution**:
```solidity
function distributeRewards(uint256 roundId) external {
    Round storage round = rounds[roundId];
    uint256 totalReward = round.fees + inflationReward;
    
    // Distribute to participants proportional to stake
    for (uint i = 0; i < round.participants.length; i++) {
        address node = round.participants[i];
        uint256 nodeStake = operators[node].stakedAmount;
        uint256 reward = (totalReward * nodeStake) / totalStaked;
        
        rewardBalances[node] += reward;
    }
}
```

**Slashing Mechanism**:
```solidity
function checkDeviation(address node, uint256 reportedValue, uint256 consensusValue) internal {
    uint256 deviation = abs(reportedValue - consensusValue) * 10000 / consensusValue;
    
    operators[node].deviationScore += deviation;
    
    // If cumulative deviation > threshold over 100 rounds
    if (operators[node].deviationScore > SLASH_THRESHOLD) {
        uint256 slashAmount = operators[node].stakedAmount / 10; // 10%
        operators[node].stakedAmount -= slashAmount;
        
        // Distribute to treasury
        treasuryBalance += slashAmount;
        
        emit Slashed(node, slashAmount, deviation);
    }
}
```

**2.4 Migration Strategy**

**Gradual Transition**:
```
Week 1-4: Parallel Operation
  • Centralized Sentinel continues as primary
  • DON runs in shadow mode (observations not used)
  • Compare DON output vs. centralized output
  • Tune consensus parameters

Week 5-8: Partial Cutover
  • 50% of users migrate to DON data
  • Monitor error rates and latency
  • Centralized Sentinel remains backup

Week 9-12: Full Cutover
  • 100% of users on DON data
  • Centralized Sentinel deprecated
  • Decommission legacy infrastructure
```

---

### Phase 3: Citadel TEE Decentralization (Q3 2026)

**Status**: ⏳ Planned

**Objective**: Decentralize credential management across multiple TEE operators

#### Challenges

**Challenge 1**: TEE hardware requirement
- Not all node operators have access to AWS Nitro or Intel SGX
- Solution: Support multiple TEE types (Nitro, SGX, SEV)

**Challenge 2**: Credential distribution
- Can't give all operators access to all credentials
- Solution: Threshold cryptography + secret sharing

#### Proposed Solution: Multi-Party Computation (MPC)

**Threshold Signature Scheme**:
```
Credential Encryption:
  • User's API key split into n shares using Shamir Secret Sharing
  • Require t-of-n shares to reconstruct (e.g., 3-of-5)
  • Each share held by different TEE operator

Order Signing Flow:
  1. Conductor requests signature from t operators
  2. Each operator (in TEE):
     - Uses their key share
     - Computes partial signature
     - Returns partial sig to Conductor
  3. Conductor combines t partial signatures
  4. Full signature reconstructed (key never fully materialized)

Security:
  • No single operator can access full key
  • Requires colluding t operators to compromise
  • TEE hardware provides additional protection per share
```

**Implementation**:
```
Phase 3a (Q3 2026): MPC Testnet
  • Deploy MPC protocol on testnet
  • 5 operator nodes (3-of-5 threshold)
  • Test with synthetic credentials
  
Phase 3b (Q4 2026): Production Rollout
  • Migrate existing credentials to MPC scheme
  • Users approve migration (re-encrypt keys)
  • Gradual rollout: 10% → 50% → 100% of users
  
Phase 3c (Q1 2027): Full Decentralization
  • All credentials managed by MPC network
  • Legacy single-TEE Citadel deprecated
  • Open operator onboarding
```

---

### Phase 4: Alchemist Prover Network (Q4 2026)

**Status**: ⏳ Planned

**Objective**: Decentralize ZK proof generation via competitive marketplace

#### Prover Marketplace Design

**Roles**:
- **Provers**: Compute ZK proofs for traders (earn fees)
- **Traders**: Request proofs, pay fees
- **Verifiers**: Validators ensure proof correctness

**Workflow**:
```
1. Proof Request
   • Trader submits: trades data + public inputs
   • Stakes escrow: 100 ZEN
   
2. Prover Bidding
   • Provers submit bids: "I'll prove for 5 ZEN in 60s"
   • Trader selects lowest bid (or fastest)
   
3. Proof Generation
   • Selected prover downloads encrypted trade data
   • Generates proof locally (Barretenberg)
   • Submits proof to smart contract
   
4. Verification
   • On-chain: HonkVerifier.verify(proof, inputs)
   • If valid: Prover receives fee from escrow
   • If invalid: Prover loses stake (slashed)
   
5. Reputation Update
   • Trader's reputation updated on-chain
   • Proof stored on IPFS (optional)
```

**Economic Model**:
```
Prover Rewards:
  • Base fee: 5-10 ZEN per proof
  • Speed bonus: 2x fee if < 30s generation
  • Volume bonus: Bulk discounts for monthly batches
  
Prover Costs:
  • Hardware: GPU server ($50-100/month)
  • Electricity: ~$20/month
  • Stake: 1,000 ZEN locked
  
Break-Even:
  • Need ~10-20 proofs/day to be profitable
  • Competitive marketplace drives fees down over time
```

---

## Fully Decentralized Architecture (2027)

**End State**:
```
┌──────────────────────────────────────────────────────────┐
│           Fully Decentralized Obscura Protocol           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │         Sentinel DON (100+ Nodes)                  │ │
│  │  • Permissionless participation                    │ │
│  │  • BFT consensus on trade data                     │ │
│  │  • ZEN staking + slashing                          │ │
│  │  • No single point of failure                      │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                                │
│                         ▼                                │
│  ┌────────────────────────────────────────────────────┐ │
│  │       Citadel MPC Network (N operators)            │ │
│  │  • Threshold signatures (t-of-n)                   │ │
│  │  • Multi-TEE support (Nitro, SGX, SEV)             │ │
│  │  • No single key holder                            │ │
│  │  • Hardware-backed security per share              │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                                │
│                         ▼                                │
│  ┌────────────────────────────────────────────────────┐ │
│  │      Alchemist Prover Network (M provers)          │ │
│  │  • Competitive proof generation marketplace        │ │
│  │  • Parallel proof computation                      │ │
│  │  • Slashing for invalid proofs                     │ │
│  │  • IPFS storage for proofs                         │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                                │
│                         ▼                                │
│  ┌────────────────────────────────────────────────────┐ │
│  │          Horizen L3 (Settlement Layer)             │ │
│  │  • Immutable reputation registry                   │ │
│  │  • On-chain proof verification                     │ │
│  │  • Governance (Gnosis Safe multi-sig)              │ │
│  │  • Staking contracts                               │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

**Properties**:
- ✅ **No centralized operators** - All services run by permissionless networks
- ✅ **Censorship resistant** - BFT consensus prevents single-actor censorship
- ✅ **Economically secure** - Staking + slashing align incentives
- ✅ **Privacy preserving** - TEE + ZK maintain confidentiality
- ✅ **Verifiable** - All data and proofs cryptographically verified
- ✅ **Scalable** - Parallel processing across network nodes

---

## Comparison: Centralized vs. Decentralized

| Aspect | Phase 1 (Centralized) | Phase 4 (Decentralized) |
|--------|----------------------|-------------------------|
| **Sentinel** | Single instance | 100+ BFT nodes |
| **Citadel** | Single TEE | MPC network (t-of-n) |
| **Alchemist** | Single prover | Competitive marketplace |
| **Trust Model** | Trust Obscura entity | Trust cryptoeconomics |
| **Censorship Resistance** | Low | High (BFT) |
| **Scalability** | Limited (single node) | High (parallel) |
| **Operational Cost** | Low ($1k/month) | Higher ($10k/month distributed) |
| **Latency** | 100ms | 200-500ms (consensus overhead) |
| **Attack Cost** | Compromise 1 server | Compromise 67% of staked nodes ($6.7M+) |

---

## Economic Model

### Token Utility ($ZEN)

**1. Staking**:
- Sentinel node operators stake 10,000 ZEN
- Citadel MPC operators stake 5,000 ZEN
- Alchemist provers stake 1,000 ZEN

**2. Fees**:
- Copy trade execution: 0.1% fee (50% to trader, 50% to protocol)
- ZK proof generation: 5-10 ZEN per proof
- Subscription: 10 ZEN/month

**3. Governance**:
- Protocol parameter changes (vote with staked ZEN)
- Slashing threshold adjustments
- Fee structure modifications

### Revenue Distribution

```
Protocol Revenue Sources:
├─ Copy Trade Fees: 50% of 0.1% per trade
├─ Subscription Fees: 10 ZEN/month per follower
└─ ZK Proof Fees: 10% of marketplace volume

Distribution:
├─ 40%: Sentinel DON operators (proportional to stake)
├─ 30%: Citadel MPC operators (per signature)
├─ 20%: Alchemist provers (per proof)
└─ 10%: Protocol treasury (development, audits)
```

---

## Conclusion

Obscura's roadmap demonstrates a **pragmatic path to decentralization**:

1. **Phase 1 (2025)**: Prove product-market fit with centralized MVP
2. **Phase 2 (2026)**: Decentralize data layer (Sentinel DON)
3. **Phase 3 (2026)**: Decentralize security layer (Citadel MPC)
4. **Phase 4 (2027)**: Decentralize compute layer (Alchemist Network)

**Final State**: A fully decentralized, permissionless, and verifiable reputation protocol for DeFi, secured by cryptoeconomics and cryptography rather than trust in any single entity.

**Learn More**:
- [Whitepaper (Full PDF)](https://drive.google.com/file/d/11idbG1itLzy922oIIy_d9BXlnVlgQEAY/view)
- [Medium Announcement](https://medium.com/@obscuraprotocol/announcing-obscura-building-the-verifiable-reputation-layer-for-defi-741de78578d8)
- [Technical Documentation](./README.md)
