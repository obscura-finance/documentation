# Zero-Knowledge Proof (ZK) Implementation

## Overview

Obscura's **Alchemist service** implements **Zero-Knowledge Succinct Non-Interactive Arguments of Knowledge (ZK-SNARKs)** to enable traders to prove their trading performance **without revealing individual trade details**. This solves the fundamental privacy-reputation paradox in copy trading.

## The Privacy-Reputation Paradox

### The Problem
- **To gain followers**: Traders must prove they are profitable
- **To prove profitability**: Traders must reveal trade details (amounts, times, strategies)
- **Revealing trades**: Exposes sensitive information and trading strategies

### The Solution: Zero-Knowledge Proofs
A trader can generate a **cryptographic proof** that:
- ✅ Proves: "I am profitable with 70% win rate"
- ❌ Does NOT reveal: Individual trades, amounts, entry/exit prices, strategies

## What is a Zero-Knowledge Proof?

A ZK proof allows a **Prover** to convince a **Verifier** that a statement is true without revealing **why** it's true.

### Properties

1. **Completeness**: If statement is true, honest prover can convince verifier
2. **Soundness**: If statement is false, malicious prover cannot convince verifier
3. **Zero-Knowledge**: Verifier learns nothing except that statement is true

### Example: Sudoku Puzzle

**Classical Proof**: Show the entire solved puzzle (reveals solution)  
**Zero-Knowledge Proof**: Prove you know the solution without showing it

```
Prover: "I solved this Sudoku"
Verifier: "Prove it without showing me the answer"
Prover: [generates ZK proof using solution as witness]
Verifier: [verifies proof] "I'm convinced you solved it"
```

## Alchemist Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Alchemist System                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │   Sentinel   │───▶│  Alchemist   │───▶│    Noir      │     │
│  │              │    │   Service    │    │   Circuit    │     │
│  │ Trade Data   │    │              │    │              │     │
│  │  Ingestion   │    │ • Validate   │    │ • Compute    │     │
│  └──────────────┘    │ • Transform  │    │   PnL        │     │
│                      │ • Format     │    │ • Verify     │     │
│                      └──────────────┘    │   Math       │     │
│                              │            │ • Generate   │     │
│                              │            │   Proof      │     │
│                              ▼            └──────────────┘     │
│                      ┌──────────────┐             │            │
│                      │    Prover    │◀────────────┘            │
│                      │  (Barretenberg)                          │
│                      │ • Generate                               │
│                      │   Witness                                │
│                      │ • Create                                 │
│                      │   Proof                                  │
│                      └──────────────┘                           │
│                              │                                   │
│                              ▼                                   │
│                      ┌──────────────┐                           │
│                      │  Smart       │                           │
│                      │  Contracts   │                           │
│                      │ (Horizen L3) │                           │
│                      │              │                           │
│                      │ • Verify     │                           │
│                      │   Proof      │                           │
│                      │ • Store      │                           │
│                      │   Report     │                           │
│                      └──────────────┘                           │
└─────────────────────────────────────────────────────────────────┘
```

## ZK Circuit Design

### Technology Stack

- **Language**: Noir (Rust-like DSL for ZK circuits)
- **Curve**: BN254 (optimal pairing-friendly curve)
- **Proof System**: Honk (PLONK-based, aggregation-friendly)
- **Prover**: Barretenberg (C++ high-performance prover)
- **Hash Function**: Poseidon2 (via noir-lang/poseidon v0.1.1)

### Circuit Inputs

#### Public Inputs (Revealed)
```rust
pub struct PublicInputs {
    trader_address: Field,        // On-chain identifier
    timestamp_start: u64,         // Report period start
    timestamp_end: u64,           // Report period end
    report_hash: Field            // Hash of computed results
}
```

#### Private Inputs (Hidden Witness)
```rust
struct PrivateInputs {
    trades: [Trade; MAX_TRADES],  // Individual trade details (max 10)
    actual_trade_count: u32       // Actual number of trades
}

struct Trade {
    side: u8,                     // 0=buy, 1=sell
    quantity: u128,               // Trade quantity (scaled by 1e12)
    price: u128,                  // Trade price (scaled by 1e12)
    fee: u128,                    // Trading fee (scaled by 1e12)
    trade_id: Field,              // Unique trade identifier
    trader_user_id: Field,        // Trader identifier
    symbol: Field,                // Hash of trading pair
    order_type: u8,               // Order type
    order_updated_at: u64,        // Trade timestamp
    exchange: Field               // Exchange identifier
}
```

### Circuit Logic

```rust
// Actual Noir implementation (simplified)

fn main(
    // Private inputs (witness)
    trades: [Trade; MAX_TRADES],
    actual_trade_count: u32,
    
    // Public inputs
    pub trader_address: Field,
    pub timestamp_start: u64,
    pub timestamp_end: u64,
    pub report_hash: Field
) -> pub TradeResult {
    // 1. Validate inputs
    assert(actual_trade_count <= MAX_TRADES);
    assert(timestamp_start < timestamp_end);
    
    // 2. Initialize multi-symbol position tracking
    let mut symbol_positions: SymbolPositions = SymbolPositions::default();
    let mut processed_trades: u32 = 0;
    
    // 3. Process each trade with queue-based matching
    for i in 0..MAX_TRADES {
        if i < actual_trade_count {
            let trade = trades[i];
            
            // Validate trade data
            assert(trade.quantity != 0);
            assert(trade.price != 0);
            
            // Calculate PnL using queue-based matching
            calculate_multi_symbol_pnl(&mut symbol_positions, trade);
            processed_trades += 1;
        }
    }
    
    // 4. Verify processed count
    assert(processed_trades == actual_trade_count);
    
    // 5. Get per-symbol PnL results
    let (symbol_pnls, symbol_count) = get_symbol_pnl_results(symbol_positions);
    
    // 6. Create cryptographic commitment
    let trade_commitment = create_trade_commitment(
        trades,
        actual_trade_count,
        trader_address,
        timestamp_start,
        timestamp_end
    );
    
    // 7. Generate and verify report hash
    let expected_report_hash = create_report_hash(
        trader_address,
        timestamp_start,
        timestamp_end,
        symbol_pnls,
        symbol_count,
        actual_trade_count
    );
    
    assert(report_hash == expected_report_hash);
    
    // 8. Return results
    TradeResult {
        symbol_pnls,
        symbol_count,
        trade_count: actual_trade_count,
        commitment: trade_commitment
    }
}
```

### Circuit Constraints

The circuit enforces:

1. **Queue-Based PnL Calculation**:
   - Buy trades are added to per-symbol queues
   - Sell trades are matched against oldest buys (FIFO)
   - PnL = (sell_price - buy_price) × quantity - fees

2. **Multi-Symbol Position Tracking**:
   - Each symbol maintains independent position queue
   - Maximum 10 lots per symbol queue
   - Realized PnL tracked per symbol

3. **Trade Validation**:
   ```
   ∀i: quantity[i] > 0 ∧ price[i] > 0
   actual_trade_count ≤ MAX_TRADES (10)
   timestamp_start < timestamp_end
   ```

4. **Report Hash Verification**:
   ```
   report_hash = H(trader_address, timestamps, symbol_pnls, trade_count)
   ```

5. **Cryptographic Commitment**:
   ```
   commitment = H(trades, actual_trade_count, trader_address, timestamps)
   ```

## Proof Generation Flow

### Step 1: Data Collection (Sentinel → Alchemist)

```json
{
  "trader_id": "0x1234...5678",
  "period_start": "2025-01-01T00:00:00Z",
  "period_end": "2025-01-31T23:59:59Z",
  "trades": [
    {
      "symbol": "BTC/USDT",
      "side": "buy",
      "entry_price": "42000.00",
      "exit_price": "43500.00",
      "quantity": "0.1",
      "fee": "4.35",
      "timestamp": "2025-01-15T10:30:00Z"
    },
    // ... up to MAX_TRADES
  ]
}
```

### Step 2: Data Transformation (Alchemist Service)

```typescript
// Transform to circuit-compatible format
const witnessData = {
  // Private inputs (witness)
  trades: trades.map(t => ({
    symbol: hashSymbol(t.symbol),
    side: t.side === 'buy' ? 0 : 1,
    entry_price: toField(t.entry_price),
    exit_price: toField(t.exit_price),
    quantity: toField(t.quantity),
    fee: toField(t.fee)
  })),
  
  // Compute public outputs from private data
  total_pnl: computeTotalPnL(trades),
  win_rate: computeWinRate(trades),
  sharpe_ratio: computeSharpeRatio(trades),
  total_trades: trades.length
};
```

### Step 3: Proof Generation (Barretenberg Prover)

```bash
# Compile circuit
nargo compile

# Generate witness
nargo execute

# Generate proof
bb prove -b ./target/circuit.json -w ./target/witness.gz -o ./proofs/proof
```

**Time Complexity**: O(n log n) where n = number of constraints  
**Proof Size**: ~192 bytes (constant, independent of number of trades)  
**Generation Time**: ~2-5 seconds for 10 trades

### Step 4: On-Chain Verification (Smart Contract)

```solidity
contract Obscura {
    IVerifier public verifier;
    address public alchemist;
    
    struct SymbolPnL {
        bytes32 symbol;
        uint256 pnlValue;
        bool pnlSign;        // true = positive, false = negative
        bool isProfitable;
    }
    
    struct TradingReport {
        address trader;
        uint64 timestampStart;
        uint64 timestampEnd;
        SymbolPnL[10] symbolPnls;
        uint256 symbolCount;
        uint256 tradeCount;
        bytes32 reportHash;
        bytes32 commitment;
    }
    
    struct ProofData {
        bytes proof;
        bytes32[] publicInputs;
    }
    
    mapping(uint256 => TradingReport) private _reports;
    mapping(address => uint256[]) private _traderReports;
    
    function submitReport(
        TradingReport calldata report,
        ProofData calldata proofData
    ) external returns (uint256 reportId) {
        // 1. Validate report data
        _validateReportData(report);
        
        // 2. Check for duplicate commitment
        require(!_usedCommitments[report.commitment], "Duplicate commitment");
        
        // 3. Verify ZK proof
        bool proofValid = verifier.verify(proofData.proof, proofData.publicInputs);
        require(proofValid, "Invalid proof");
        
        // 4. Verify public inputs match report
        _verifyPublicInputs(report, proofData.publicInputs);
        
        // 5. Store report
        reportId = ++_reportCounter;
        _reports[reportId] = report;
        _traderReports[report.trader].push(reportId);
        _usedCommitments[report.commitment] = true;
        
        // 6. Update trader symbols and stats
        _updateTraderSymbols(report.trader, report.symbolPnls, report.symbolCount);
        
        emit ReportSubmitted(
            report.trader,
            reportId,
            report.timestampStart,
            report.timestampEnd,
            report.symbolCount,
            report.tradeCount
        );
    }
}
```

## Cryptographic Security

### BN254 Elliptic Curve

**Curve Equation**: y² = x³ + 3  
**Field Size**: 254 bits (~76 decimal digits)  
**Security Level**: ~128-bit security  
**Pairing-Friendly**: Optimal Type III pairing

**Why BN254?**
- ✅ Fast pairing operations (needed for SNARK verification)
- ✅ Ethereum precompile support (gas-efficient verification)
- ✅ Well-studied and battle-tested
- ✅ Supported by major ZK frameworks (Noir, Circom, Halo2)

### Poseidon2 Hash Function

**Purpose**: Cryptographic commitment and report hash generation  
**Implementation**: noir-lang/poseidon v0.1.1 (Poseidon2Hasher)  
**Design**: Sponge construction optimized for ZK circuits  
**Security**: 128-bit collision resistance

**Why Poseidon2?**
- ✅ 10x fewer constraints than SHA256 in ZK circuits
- ✅ Native field arithmetic (no bitwise operations)
- ✅ Designed specifically for ZK-SNARKs
- ✅ Used for HashMap key hashing in circuit

```
Poseidon Hash Comparison:
┌──────────┬───────────┬──────────────┐
│ Function │ Constraints│ Proving Time│
├──────────┼───────────┼──────────────┤
│ SHA256   │ ~27,000   │ ~500ms      │
│ Poseidon │ ~300      │ ~5ms        │
└──────────┴───────────┴──────────────┘
```

### Honk Proof System

**Type**: PLONK-based with aggregation support  
**Trusted Setup**: Universal (one-time ceremony, circuit-agnostic)  
**Proof Size**: ~192 bytes  
**Verification Gas**: ~280k gas on Ethereum

**Advantages**:
- ✅ Universal trusted setup (reusable across circuits)
- ✅ Constant proof size
- ✅ Fast verification
- ✅ Proof aggregation (batch verify multiple proofs)

## Performance Metrics

### Proof Generation

| Trades | Constraints | Proof Time | Memory |
|--------|-------------|------------|--------|
| 2      | ~12,000     | ~150ms     | 256MB  |
| 3      | ~15,000     | ~200ms     | 512MB  |
| 5      | ~20,000     | ~300ms     | 512MB  |
| 10     | ~30,000     | ~500ms     | 1GB    |

### Proof Verification

| Location | Time | Cost |
|----------|------|------|
| Off-chain (CPU) | ~5ms | Free |
| Horizen Testnet | ~50ms | ~$0.001 gas |
| Ethereum | ~200ms | ~$15 gas (280k @ 50 gwei) |

### Scalability

**Current**: 10 trades per proof (MAX_TRADES = 10)  
**Precision**: 1e12 scaling factor (SCALE_FACTOR)  
**Queue Size**: 10 lots per symbol (MAX_QUEUE_SIZE)  
**Throughput**: 100+ proofs/hour (single prover instance)

## Deployed System (Horizen Testnet)

### Network Details

- **Name**: Horizen Testnet
- **Chain ID**: 845320009
- **RPC**: https://horizen-rpc-testnet.appchain.base.org
- **Explorer**: https://horizen-explorer-testnet.appchain.base.org

### Contract Addresses

**HonkVerifier**: `0xBCfD5cf1255C839441D400D5a31De7e625f00095`
- Verifies Honk proofs on-chain
- Automatically generated by Noir compiler
- Gas-optimized assembly implementation
- [View on Explorer](https://horizen-explorer-testnet.appchain.base.org/address/0xBCfD5cf1255C839441D400D5a31De7e625f00095)

**Obscura**: `0xC9483BFE9806788169ac0791bb149F0cD7d43125`
- Main reputation contract
- Stores trader performance reports
- Emits events for indexing
- [View on Explorer](https://horizen-explorer-testnet.appchain.base.org/address/0xC9483BFE9806788169ac0791bb149F0cD7d43125)

**Deployer/Alchemist**: `0x3B465D0695621f330a45BCc15fcF6B2d8f2046d6`

### Deployment Details
- **Date**: October 21, 2025 (03:19:31 UTC)
- **Block Number**: 20932983

## Privacy Guarantees

### What is Hidden (Zero-Knowledge)

✅ Individual trade details:
- Trade prices
- Trade quantities
- Trading fees
- Trade IDs and timestamps
- Order types
- Exchange identifiers

✅ Trading strategies:
- Order of trades
- Position sizing logic
- Entry/exit criteria
- Symbol selection
- Risk management rules

### What is Revealed (Public Outputs)

📊 Minimal public information:
- Trader address
- Report time period (start/end timestamps)
- Report hash (cryptographic commitment to results)
- Per-symbol PnL (via TradeResult)
- Total number of trades

### Attack Resistance

**Replay Attack**: Prevented by unique report hash and timestamp validation  
**Front-Running**: Public outputs don't help predict future trades  
**Data Tampering**: Cryptographic commitment ensures data integrity  
**Invalid Proofs**: HonkVerifier rejects any invalid ZK proofs

## Integration Example

### Submit Performance Report

```typescript
import { AlchemistClient } from '@obscura/alchemist-sdk';

const alchemist = new AlchemistClient({
  apiUrl: 'https://alchemist.obscura.finance',
  apiKey: process.env.ALCHEMIST_API_KEY
});

// 1. Fetch trade data from Sentinel
const trades = await sentinel.getTrades({
  traderId: '0x1234...5678',
  startDate: '2025-01-01',
  endDate: '2025-01-31'
});

// 2. Generate ZK proof
const { proof, publicInputs } = await alchemist.generateProof({
  trades,
  traderAddress: '0x1234...5678'
});

// 3. Submit to blockchain
const tx = await obscuraContract.submitReport(proof, publicInputs);
await tx.wait();

console.log('Performance report submitted:', tx.hash);
```

### Query Trader Reputation

```typescript
const reports = await obscuraContract.getReports('0x1234...5678');

reports.forEach(report => {
  console.log(`
    Total PnL: ${report.totalPnL}
    Win Rate: ${report.winRate}%
    Sharpe Ratio: ${report.sharpeRatio / 100}
    Total Trades: ${report.totalTrades}
    Timestamp: ${new Date(report.timestamp * 1000)}
  `);
});
```

## Future Enhancements

### Recursive Proofs (Roadmap Q3 2026)
- Aggregate multiple monthly reports into single annual proof
- Constant verification cost regardless of time period

### Decentralized Prover Network (Roadmap Q4 2026)
- Competitive marketplace for proof generation
- Stake-based slashing for invalid proofs
- Parallel proof generation for faster throughput

### Cross-Exchange Aggregation (Roadmap Q1 2027)
- Single proof for trades across multiple exchanges
- Unified reputation score

## References

- **Noir Documentation**: https://noir-lang.org/
- **Barretenberg**: https://github.com/AztecProtocol/barretenberg
- **BN254 Curve**: https://eprint.iacr.org/2010/429
- **Poseidon Hash**: https://eprint.iacr.org/2019/458
- **PLONK**: https://eprint.iacr.org/2019/953
- **Horizen**: https://www.horizen.io/
