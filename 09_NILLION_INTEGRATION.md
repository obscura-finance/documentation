# Nillion Integration

## Overview

Obscura integrates Nillion's privacy-preserving infrastructure (nilCC + nilDB) to strengthen its confidential compute and data custody model. **Phase 1 (nilCC) is complete** — Citadel now runs on Nillion nilCC with AMD SEV-SNP hardware attestation. Phases 2-3 (nilDB + NUC access control) are in planning.

## Component Mapping

| Current                          | Target               | Function                                      | Status          |
|----------------------------------|----------------------|-----------------------------------------------|-----------------|
| Citadel (Nillion nilCC)          | *(Complete)*         | Decrypt credentials, sign orders, compute     | ✅ Live          |
| Encrypted fields in Postgres     | **nilDB**            | Secret-share API keys, PII, alpha content     | ⏳ Planned       |
| Custom access control            | **NUCs**             | Programmatic grant/revoke per subscriber      | ⏳ Planned       |
| (Optional) CEX verification      | **Primus zkTLS**     | Binance PnL attestation                       | ⏳ Planned       |

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Obscura Services                            │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────┐     ┌──────────┐     ┌──────────┐                    │
│  │ Sentinel │     │Conductor │     │Alchemist │                    │
│  └────┬─────┘     └────┬─────┘     └────┬─────┘                    │
│       │                │                │                           │
│       └────────────────┼────────────────┘                           │
│                        │                                            │
│                        ▼                                            │
│              ┌─────────────────┐                                    │
│              │  Citadel API    │  (unchanged contract)              │
│              └────────┬────────┘                                    │
└───────────────────────┼─────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Nillion Infrastructure                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌───────────────────────────────────────────────────────────┐    │
│   │                      nilCC (TEE)                          │    │
│   │  • Decrypt credentials                                    │    │
│   │  • Sign exchange orders                                   │    │
│   │  • Compute performance metrics                            │    │
│   │  • Reconstruct secrets from nilDB                         │    │
│   └───────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    nilDB (3 nodes)                          │  │
│   │  ┌─────────┐      ┌─────────┐      ┌─────────┐             │  │
│   │  │ Share 1 │      │ Share 2 │      │ Share 3 │             │  │
│   │  │ (Node A)│      │ (Node B)│      │ (Node C)│             │  │
│   │  └─────────┘      └─────────┘      └─────────┘             │  │
│   │  (no single node holds plaintext)                          │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Data Flow

### Credential Decryption (Copy Trade Execution)

```
1. Conductor requests credential decrypt via Citadel API
2. Citadel (nilCC) fetches shares from nilDB nodes
3. Shares reconstructed inside nilCC (TEE boundary)
4. Order signed with plaintext credentials
5. Signed payload returned; plaintext purged
```

### Secret Storage

```
1. Sentinel receives API credentials from user
2. Citadel encrypts + secret-shares via nilCC
3. Shares distributed to 3 nilDB nodes
4. Postgres stores opaque reference (nildb_ref)
```

### Alpha/Signal Access (Phase 3)

```
1. Trader publishes alpha → nilCC secret-shares → nilDB
2. Follower subscribes (on-chain payment verified)
3. nilCC grants NUC permission to follower
4. Follower reads: shares reconstructed in nilCC → delivered
5. Subscription expires → permission auto-revoked
```

## Integration Phases

| Phase | Scope                                    | Timeline   |
|-------|------------------------------------------|------------|
| **1** | Citadel → nilCC (decrypt + sign)         | ✅ **Complete** |
| **2** | Postgres secrets → nilDB                 | Weeks 5–8  |
| **3** | Alpha content + leaderboard metrics      | Weeks 9–12 |

## Security Model Changes

| Threat Scenario       | Previous (Evervault)             | Current (Nillion nilCC)                  |
|-----------------------|----------------------------------|------------------------------------------|
| **DB breach**         | Encrypted blobs exposed          | Versioned nil: encrypted; master key in TEE |
| **Infra compromise**  | Trust AWS + Evervault            | Bare-metal TEE; decentralized infra      |
| **Insider access**    | Single encrypted store           | Master key isolated in AMD SEV-SNP       |
| **Service compromise**| Temp plaintext in memory         | Same (TEE isolation)                     |

## Performance Considerations

- **Current latency**: Citadel ~50ms, Conductor e2e 200–500ms
- **nilCC overhead**: Additional network hops for nilDB reconstruction
- **Mitigations**:
  - Session-based secret caching inside nilCC (short TTL)
  - Pre-signed order templates for common operations
  - Async reconstruction for non-critical paths

## API Compatibility

Citadel endpoints remain unchanged:

```
POST /encrypt       # nilCC encrypt + store shares
POST /decrypt       # nilCC reconstruct from nilDB
POST /sign-order    # nilCC sign with decrypted creds
GET  /health        # nilCC health + nilDB quorum check
```

Conductor and Sentinel require **no changes** in Phase 1.

## Dependencies

- Nillion SDK (nilCC + nilDB clients)
- 3 nilDB node operators (self-hosted or Nillion-managed)
- Optional: Primus Labs zkTLS for CEX verification

---

## Tickr Parity: Comprehensive Feature Plan

With Tickr sunsetted, Obscura is positioned to continue that vision. This section outlines the gaps and implementation plan to achieve full parity and beyond.

### Gap Analysis

| Feature | Tickr | Obscura Current | Gap |
|---------|-------|-----------------|-----|
| CEX verification | zkTLS (Primus) | API key trust | No cryptographic proof |
| Credential flow | Browser extension | API key submission | Higher friction + trust burden |
| Access control | NUCs (on-chain) | Application-level | No trustless pipeline |
| Operator blindness | Full (nilCC) | **nilCC (AMD SEV-SNP)** | ✅ Achieved with migration |
| DEX integration | Hyperliquid native | CCXT adapters | No privacy-first DEX flow |

---

### Flow 1: zkTLS CEX Verification (Primus Labs)

**Goal**: Traders prove CEX performance without sharing API keys.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     zkTLS Verification Flow                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────┐                                                   │
│   │   Trader    │                                                   │
│   │   Browser   │                                                   │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          │ 1. Install Primus Extension                              │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │   Primus    │                                                   │
│   │  Extension  │                                                   │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          │ 2. Login to Binance (user's browser)                     │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │   Binance   │                                                   │
│   │   Session   │                                                   │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          │ 3. Extension extracts PnL data                           │
│          │    Generates zkTLS proof                                 │
│          ▼                                                          │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │              Encrypted + Signed Payload                     │  │
│   │  • PnL (24h, 7d, 30d, 90d)                                  │  │
│   │  • Trade count                                              │  │
│   │  • Win rate                                                 │  │
│   │  • zkProof signature                                        │  │
│   └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                      │
│                              │ 4. Forward to Obscura                │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    Obscura Frontend                         │  │
│   └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                      │
│                              │ 5. Send payload to backend           │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                      nilCC (TEE)                            │  │
│   │  • Verify zkTLS signature                                   │  │
│   │  • Decrypt payload                                          │  │
│   │  • Compute reputation metrics                               │  │
│   │  • Secret-share to nilDB                                    │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   RESULT: Verified performance WITHOUT API keys                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Implementation**:

| Component | Work Required |
|-----------|---------------|
| Frontend | Add Primus extension detection + zkTLS flow |
| API Gateway | New endpoint: `POST /verify/zktls` |
| Citadel (nilCC) | zkTLS signature verification + payload decryption |
| Database | New table: `zktls_verifications` |

**Primus Integration Points**:
- SDK: `@primus-labs/zktls-sdk`
- Templates: Binance PnL, OKX PnL, Bybit PnL
- Extension: Chrome Web Store (user installs)

---

### Flow 2: Dual Verification Paths

**Goal**: Support both API key (for copy trading) and zkTLS (for verification-only).

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Trader Onboarding Flow                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                     ┌──────────────────┐                            │
│                     │  Connect Exchange │                           │
│                     └────────┬─────────┘                            │
│                              │                                      │
│              ┌───────────────┼───────────────┐                      │
│              │               │               │                      │
│              ▼               ▼               ▼                      │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│   │  zkTLS Only  │  │   API Key    │  │    Wallet    │             │
│   │ (Verify PnL) │  │ (Copy Trade) │  │ (DEX/Onchain)│             │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│          │                 │                 │                      │
│          ▼                 ▼                 ▼                      │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│   │ Primus Ext.  │  │ Citadel API  │  │ WalletConnect│             │
│   │ + nilCC      │  │ + nilDB      │  │ + nilCC      │             │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│          │                 │                 │                      │
│          └─────────────────┼─────────────────┘                      │
│                            │                                        │
│                            ▼                                        │
│              ┌─────────────────────────┐                            │
│              │   Unified OBSCURA_ID    │                            │
│              │   + Verified Metrics    │                            │
│              └─────────────────────────┘                            │
│                                                                     │
│   USER CHOICE:                                                      │
│   • zkTLS: "I just want to prove my track record"                   │
│   • API Key: "I want followers to copy my trades"                   │
│   • Wallet: "I trade on-chain (DEX)"                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Benefits**:
- Lower barrier for verification-only traders
- API keys only needed for copy execution
- "Never share your keys" marketing angle

---

### Flow 3: On-Chain Subscriptions + NUC Access Control

**Goal**: Trustless payment → access pipeline for premium alpha.

```
┌─────────────────────────────────────────────────────────────────────┐
│                 On-Chain Subscription Flow                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   TRADER PUBLISHES ALPHA                                            │
│   ──────────────────────                                            │
│                                                                     │
│   ┌──────────┐     ┌──────────┐     ┌──────────────────────────┐   │
│   │  Trader  │ ──► │  nilCC   │ ──► │       nilDB              │   │
│   │  writes  │     │  (TEE)   │     │  (secret-shared alpha)   │   │
│   │  alpha   │     │          │     │                          │   │
│   └──────────┘     └──────────┘     └──────────────────────────┘   │
│                                                                     │
│   FOLLOWER SUBSCRIBES                                               │
│   ───────────────────                                               │
│                                                                     │
│   ┌──────────┐     ┌──────────────┐     ┌──────────┐               │
│   │ Follower │ ──► │  Payment     │ ──► │ nilChain │               │
│   │  clicks  │     │  (USDC/ETH)  │     │   txn    │               │
│   │  "Sub"   │     │              │     │          │               │
│   └──────────┘     └──────────────┘     └────┬─────┘               │
│                                              │                      │
│                                              │ txn hash             │
│                                              ▼                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                      nilCC (TEE)                            │  │
│   │  1. Verify payment txn on-chain                             │  │
│   │  2. Check amount matches subscription tier                  │  │
│   │  3. Grant NUC (Nillion User Credential) to follower         │  │
│   │  4. Set expiration (30 days, etc.)                          │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   FOLLOWER VIEWS ALPHA                                              │
│   ────────────────────                                              │
│                                                                     │
│   ┌──────────┐     ┌──────────┐     ┌──────────────────────────┐   │
│   │ Follower │ ──► │  nilCC   │ ──► │       nilDB              │   │
│   │ requests │     │  checks  │     │  reconstruct shares      │   │
│   │  alpha   │     │   NUC    │     │  if NUC valid            │   │
│   └──────────┘     └──────────┘     └──────────────────────────┘   │
│                          │                                          │
│                          │ NUC expired?                             │
│                          ▼                                          │
│                    ┌───────────┐                                    │
│                    │  DENIED   │                                    │
│                    │ (auto-    │                                    │
│                    │  revoked) │                                    │
│                    └───────────┘                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Smart Contract Requirements**:

```solidity
// SubscriptionRegistry.sol (simplified)
contract SubscriptionRegistry {
    mapping(address => mapping(address => uint256)) public subscriptions;
    // subscriber => trader => expiration timestamp
    
    event Subscribed(address indexed subscriber, address indexed trader, uint256 expires);
    
    function subscribe(address trader, uint256 months) external payable {
        require(msg.value >= traders[trader].price * months);
        subscriptions[msg.sender][trader] = block.timestamp + (months * 30 days);
        emit Subscribed(msg.sender, trader, subscriptions[msg.sender][trader]);
    }
}
```

**nilCC Verification**:
```python
async def verify_subscription(txn_hash: str, follower: str, trader: str) -> bool:
    # 1. Fetch txn from chain
    txn = await chain.get_transaction(txn_hash)
    
    # 2. Decode subscription event
    event = decode_subscription_event(txn)
    
    # 3. Verify matches request
    if event.subscriber != follower or event.trader != trader:
        return False
    
    # 4. Check not expired
    if event.expires < time.now():
        return False
    
    # 5. Grant NUC
    await nildb.grant_permission(follower, trader_alpha_ref)
    return True
```

---

### Flow 4: Hyperliquid Native Integration

**Goal**: Privacy-first DEX flow matching Tickr's Hyperliquid integration.

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Hyperliquid Integration Flow                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────┐                                                      │
│   │  Trader  │                                                      │
│   │  Wallet  │                                                      │
│   └────┬─────┘                                                      │
│        │                                                            │
│        │ 1. Connect via WalletConnect/Injected                      │
│        ▼                                                            │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                   Obscura Frontend                          │  │
│   │  • Sign message proving wallet ownership                    │  │
│   │  • No private key exposure                                  │  │
│   └─────────────────────────────────────────────────────────────┘  │
│        │                                                            │
│        │ 2. Signed message + wallet address                         │
│        ▼                                                            │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                      nilCC (TEE)                            │  │
│   │                                                             │  │
│   │  3. Verify signature                                        │  │
│   │                                                             │  │
│   │  4. Fetch Hyperliquid data (inside TEE):                    │  │
│   │     • GET /info (account state)                             │  │
│   │     • GET /userFills (trade history)                        │  │
│   │     • GET /clearinghouseState (positions)                   │  │
│   │                                                             │  │
│   │  5. Compute metrics:                                        │  │
│   │     • PnL (realized + unrealized)                           │  │
│   │     • Win rate, Sharpe, drawdown                            │  │
│   │     • Volume, trade count                                   │  │
│   │                                                             │  │
│   │  6. Secret-share to nilDB                                   │  │
│   │     (wallet address NEVER leaves TEE in plaintext)          │  │
│   │                                                             │  │
│   └─────────────────────────────────────────────────────────────┘  │
│        │                                                            │
│        ▼                                                            │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │   Public Leaderboard shows:                                 │  │
│   │   • OBSCURA_ID (not wallet)                                 │  │
│   │   • Verified metrics                                        │  │
│   │   • "Hyperliquid" badge                                     │  │
│   │   • Proof hash                                              │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   PRIVACY GUARANTEE:                                                │
│   • Wallet address known only inside nilCC                          │
│   • Followers see performance, not wallet                           │
│   • No front-running possible                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Hyperliquid API Endpoints** (called from nilCC):

| Endpoint | Data | Use |
|----------|------|-----|
| `POST /info` | Account state | Equity, margin |
| `POST /info` (userFills) | Trade history | Win rate, volume |
| `POST /info` (clearinghouse) | Open positions | Unrealized PnL |

---

### Implementation Roadmap

| Phase | Duration | Deliverables |
|-------|----------|--------------|
| **1A** | Week 1–2 | Primus SDK integration spike |
| **1B** | Week 3–4 | zkTLS verification endpoint + frontend flow |
| **2A** | Week 5–6 | Hyperliquid nilCC integration |
| **2B** | Week 7–8 | Wallet-connect flow + signature verification |
| **3A** | Week 9–10 | Smart contract: SubscriptionRegistry |
| **3B** | Week 11–12 | NUC-based alpha access in nilCC |
| **4** | Week 13–14 | "Operator-blind" attestation + marketing |

---

### Success Metrics

| Metric | Target | Rationale |
|--------|--------|-----------|
| zkTLS verifications/week | 100+ | Adoption of no-API-key flow |
| Hyperliquid traders | 50+ | DEX coverage |
| On-chain subscriptions | 25+ | Trustless monetization |
| "Operator-blind" mentions | 10+ articles | Trust narrative penetration |

---

*Last Updated: February 2026*
