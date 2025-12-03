# B2B Integration Guide

## Overview

This guide explains how **B2B platforms** (trading bots, signal providers, copy trading aggregators) can integrate with Obscura's copy trading infrastructure to offer white-labeled copy trading to their users.

---

## 🎯 Who Is This For?

| Platform Type | Use Case |
|---------------|----------|
| **Trading Bots** | Automatically copy bot trades to subscribers |
| **Signal Providers** | Distribute trade signals to followers |
| **Prop Trading Firms** | Mirror trades across managed accounts |
| **Social Trading Apps** | Build copy trading features on your platform |
| **DeFi Aggregators** | Cross-exchange trade replication |

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Your Platform (B2B Tenant)                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Trading Bot │  │ Signal App  │  │ Dashboard   │              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
└─────────┼────────────────┼────────────────┼─────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Obscura API Gateway                          │
│         POST /v2/pools/{pool_id}/signals                         │
│         (Tenant-Isolated, Rate-Limited, Priority Queued)         │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   B2B Signal Processing                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Idempotency  │  │  Priority    │  │  Rate        │          │
│  │   Check      │  │  Queueing    │  │  Limiting    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                 Copy Trade Execution Engine                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  For each subscriber:                                     │   │
│  │  1. Check subscription mode (Fixed Amount / Advanced)     │   │
│  │  2. Calculate position size based on risk parameters      │   │
│  │  3. Apply Martingale scaling (if enabled)                 │   │
│  │  4. Decrypt exchange credentials (via Citadel TEE)        │   │
│  │  5. Execute order on subscriber's exchange account        │   │
│  │  6. Track P&L for profitability reporting                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📋 Getting Started

### Step 1: Tenant Onboarding

Contact Obscura to register as a B2B tenant. You'll receive:

| Credential | Description |
|------------|-------------|
| `tenant_id` | Your unique organization identifier (UUID) |
| `api_key` | API key with prefix `obsc_live_...` or `obsc_test_...` |
| `scopes` | Permissions: `signal:write`, `pool:manage`, `stream:access` |

```bash
# Example API key header
X-API-Key: obsc_live_abc123xyz789...
```

### Step 2: Create a Trading Pool

Pools are containers for trading strategies. All signals you submit go to a specific pool, and users subscribe to pools.

```bash
POST /v2/pools
Content-Type: application/json
X-API-Key: obsc_live_...

{
    "name": "AlgoBot Pro Strategy",
    "description": "Momentum-based BTC/ETH trading bot with 65% win rate",
    "pool_type": "partner",
    "risk_class": "medium",
    "traders": [
        {
            "trader_id": "your-bot-account-uuid",
            "weight": 100
        }
    ]
}
```

**Response:**
```json
{
    "id": "pool-uuid-123",
    "name": "AlgoBot Pro Strategy",
    "status": "draft",
    "tenant_id": "your-tenant-uuid",
    "fee_percentage": 15.0,
    "platform_take_percentage": 3.0
}
```

### Step 3: Activate the Pool

```bash
POST /v2/pools/{pool_id}/activate
X-API-Key: obsc_live_...
```

### Step 4: Users Subscribe to Your Pool

Your users subscribe through your UI, which calls our API:

**Fixed Amount Mode** (Beginner-Friendly):
```json
POST /v2/pools/{pool_id}/subscribe

{
    "subscription_mode": "fixed_amount",
    "total_investment_usd": 1000,
    "max_loss_usd": 100,
    "profit_target_usd": 200,
    "daily_loss_limit_usd": 50
}
```

**Advanced Mode** (Experienced Traders):
```json
{
    "subscription_mode": "advanced",
    "per_pair_config": {
        "BTCUSDT": {
            "amount_usd": 500,
            "stop_loss_pct": 5,
            "take_profit_pct": 15
        },
        "ETHUSDT": {
            "amount_usd": 300,
            "stop_loss_pct": 7,
            "take_profit_pct": 20
        }
    },
    "enable_martingale": true,
    "martingale_multiplier": 1.5,
    "martingale_max_levels": 3,
    "daily_loss_limit_usd": 50
}
```

---

## 🚀 Submitting Trade Signals

### Option A: REST API (Recommended for < 100 signals/min)

When your trading bot executes a trade, submit it as a signal:

```bash
POST /v2/pools/{pool_id}/signals
Content-Type: application/json
X-API-Key: obsc_live_...

{
    "symbol": "BTCUSDT",
    "side": "buy",
    "quantity": 0.5,
    "order_type": "market",
    "exchange": "binance",
    "platform": "futures",
    "leverage": 10,
    "stop_loss_pct": 5,
    "take_profit_pct": 15,
    "priority": "high",
    "external_trade_id": "bot-trade-12345",
    "metadata": {
        "strategy": "momentum_breakout",
        "confidence": 0.85
    }
}
```

**Response (202 Accepted):**
```json
{
    "signal_id": "sig_a1b2c3d4e5f6",
    "pool_id": "pool-uuid-123",
    "status": "queued",
    "queued_at": "2025-12-03T10:30:00Z",
    "priority": "high",
    "estimated_subscribers": 150,
    "idempotency_key": "idem:abc123..."
}
```

### Option B: Batch Submission

Submit multiple signals at once (max 100 per request):

```bash
POST /v2/pools/{pool_id}/signals/bulk

{
    "signals": [
        {"symbol": "BTCUSDT", "side": "buy", "quantity": 0.5, ...},
        {"symbol": "ETHUSDT", "side": "buy", "quantity": 2.0, ...},
        {"symbol": "SOLUSDT", "side": "sell", "quantity": 10, ...}
    ],
    "batch_priority": "high",
    "fail_fast": false
}
```

### Option C: High-Volume Stream (> 100 signals/min)

For high-frequency trading bots, register for direct Redis Stream access:

```bash
POST /v2/pools/{pool_id}/streams/register

{
    "expected_signals_per_minute": 500
}
```

**Response:**
```json
{
    "stream_id": "stream_abc123",
    "stream_config": {
        "stream_name": "stream:trades:cell-0:tenant123:pool456",
        "consumer_group": "cg_pool456",
        "max_batch_size": 50
    },
    "credentials": {
        "stream_url": "rediss://...",
        "auth_token": "xyz...",
        "expires_at": "2025-12-04T10:30:00Z"
    },
    "publish_format": {
        "fields": {
            "payload": "JSON-encoded signal",
            "priority": "low|normal|high|critical",
            "idempotency_key": "unique key"
        },
        "example_command": "XADD stream:trades:cell-0:... * payload '{...}'"
    }
}
```

---

## 📊 Signal Parameters Reference

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `symbol` | string | Trading pair (e.g., "BTCUSDT", "ETH/USDT") |
| `side` | enum | `buy` or `sell` |
| `quantity` | float | Base asset quantity |

### Optional Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `order_type` | enum | `market` | `market` or `limit` |
| `price` | float | null | Required for limit orders |
| `exchange` | string | `binance` | Target exchange |
| `platform` | string | `spot` | `spot`, `futures`, `margin` |
| `leverage` | int | null | 1-125 for futures |
| `margin_mode` | string | null | `isolated` or `cross` |
| `stop_loss_pct` | float | null | Stop loss percentage |
| `take_profit_pct` | float | null | Take profit percentage |
| `priority` | enum | `normal` | `low`, `normal`, `high`, `critical` |
| `execute_immediately` | bool | false | Skip queue, sync execution |
| `external_trade_id` | string | null | For idempotency |
| `source_trader_id` | UUID | null | If from registered trader |
| `external_trader_ref` | string | null | Your internal reference |
| `metadata` | object | null | Custom data (strategy, confidence, etc.) |

### Priority Levels

| Priority | Use Case | Queue Behavior |
|----------|----------|----------------|
| `critical` | Emergency exits, liquidation prevention | Immediate execution, skip queue |
| `high` | Important signals, high confidence | Front of queue |
| `normal` | Standard trading signals | FIFO processing |
| `low` | Batch processing, non-urgent | Back of queue |

---

## 🔍 Tracking Signal Status

```bash
GET /v2/pools/{pool_id}/signals/{signal_id}
```

**Response:**
```json
{
    "signal_id": "sig_a1b2c3d4e5f6",
    "pool_id": "pool-uuid-123",
    "status": "completed",
    "priority": "high",
    "created_at": "2025-12-03T10:30:00Z",
    "updated_at": "2025-12-03T10:30:05Z",
    "total_subscribers": 150,
    "executions_pending": 0,
    "executions_completed": 148,
    "executions_failed": 2,
    "queue_wait_ms": 50,
    "avg_execution_ms": 180,
    "errors": [
        {
            "subscriber_id": "user-uuid-1",
            "error": "Insufficient balance",
            "timestamp": "2025-12-03T10:30:04Z"
        }
    ]
}
```

### Signal Status Values

| Status | Description |
|--------|-------------|
| `queued` | Signal received, waiting in priority queue |
| `processing` | Being executed across subscribers |
| `completed` | All executions finished |
| `failed` | Signal failed (validation error, etc.) |
| `partial` | Some executions failed |

---

## 🏦 Supported Exchanges

Obscura supports **100+ exchanges** via CCXT, with certified support for major CEX and DEX platforms.

### Certified CEX (Fully Tested)

| Exchange | Spot | Futures | Margin | WebSocket |
|----------|:----:|:-------:|:------:|:---------:|
| Binance | ✅ | ✅ | ✅ | ✅ |
| Binance USDⓈ-M | - | ✅ | - | ✅ |
| Binance COIN-M | - | ✅ | - | ✅ |
| Bybit | ✅ | ✅ | ✅ | ✅ |
| OKX | ✅ | ✅ | ✅ | ✅ |
| Gate.io | ✅ | ✅ | ✅ | ✅ |
| KuCoin | ✅ | ✅ | ✅ | ✅ |
| Bitget | ✅ | ✅ | ✅ | ✅ |
| MEXC | ✅ | ✅ | - | ✅ |
| HTX (Huobi) | ✅ | ✅ | ✅ | ✅ |
| BingX | ✅ | ✅ | - | ✅ |
| Crypto.com | ✅ | - | - | ✅ |
| BitMart | ✅ | ✅ | - | ✅ |
| CoinEx | ✅ | ✅ | - | ✅ |
| WOO X | ✅ | ✅ | - | ✅ |

### Certified DEX

| Exchange | Type | Chains | WebSocket |
|----------|------|--------|:---------:|
| Hyperliquid | Perps DEX | Arbitrum | ✅ |
| dYdX | Perps DEX | dYdX Chain | ✅ |
| Paradex | Perps DEX | StarkNet | ✅ |
| Apex Pro | Perps DEX | StarkEx | ✅ |
| WOOFi Pro | DEX | Multi-chain | ✅ |

### Supported (Beta)

50+ additional exchanges are supported via CCXT including:
- Coinbase, Kraken, Bitstamp (US-friendly)
- Phemex, Deribit (derivatives)
- Gemini, BitFlyer (regional)

Full list available via:
```bash
GET /exchanges/supported
```

---

## 🔒 Security & Tenant Isolation

### Data Isolation

| Data Type | Isolation Level |
|-----------|-----------------|
| Pools | Tenant-scoped (only your users see your pools) |
| Subscriptions | User + Tenant scoped |
| Signals | Tenant-scoped |
| Credentials | TEE-encrypted, user-scoped |
| Trades | User-scoped |

### API Security

- **TLS 1.3**: All API traffic encrypted
- **API Key Authentication**: Tenant-scoped keys with permission scopes
- **Rate Limiting**: Configurable per-tenant limits
- **IP Allowlisting**: Optional (contact support)

### Credential Protection

All exchange API credentials are:
1. Encrypted with AES-256-GCM before storage
2. Decrypted only inside AWS Nitro Enclaves (TEE)
3. Never exposed in plaintext, even to Obscura operators

---

## 📈 Subscription Modes Explained

### Fixed Amount Mode

**Best for**: Beginners, simple risk management

Users set three plain-language parameters:
- **Total Investment**: How much to allocate to this pool
- **Max Loss**: "How much are you ready to lose?"
- **Profit Target**: "How much profit before stopping?"

**Position Sizing Logic**:
```
position_size = total_investment / 10  # Divide across ~10 positions
if remaining_loss_tolerance < position_size:
    position_size = remaining_loss_tolerance
```

### Advanced Mode

**Best for**: Experienced traders, custom risk management

Users configure per-pair settings:
```json
{
    "BTCUSDT": {"amount_usd": 500, "stop_loss_pct": 5, "take_profit_pct": 15},
    "ETHUSDT": {"amount_usd": 300, "stop_loss_pct": 7, "take_profit_pct": 20}
}
```

Optional features:
- **Martingale**: Increase position size after losses
- **Global Limits**: Override per-pair settings
- **Daily Loss Limit**: Auto-pause after daily limit hit

---

## 💰 Fee Structure

| Fee Type | Percentage | Recipient |
|----------|------------|-----------|
| Profit Share | 10-20% | Pool creator (you) |
| Platform Fee | 3% | Obscura |

Fees are charged only on **profitable trades**. No fees on losing trades.

---

## 🔧 Error Handling

### Common Error Codes

| Code | Description | Solution |
|------|-------------|----------|
| `400` | Invalid signal format | Check required fields |
| `401` | Invalid API key | Check X-API-Key header |
| `403` | Insufficient permissions | Check API key scopes |
| `404` | Pool not found | Verify pool_id |
| `409` | Duplicate signal | Signal already processed (idempotency) |
| `429` | Rate limit exceeded | Reduce request frequency |
| `503` | Service unavailable | Retry with exponential backoff |

### Idempotency

Signals with the same `external_trade_id` are deduplicated:

```bash
# First request - creates signal
POST /v2/pools/{pool_id}/signals
{"external_trade_id": "bot-123", ...}
# Response: 202 Accepted, signal_id: "sig_abc"

# Second request - returns existing signal
POST /v2/pools/{pool_id}/signals
{"external_trade_id": "bot-123", ...}
# Response: 200 OK, signal_id: "sig_abc" (same signal, no duplicate execution)
```

---

## 🛠️ SDK Examples

### Python

```python
import httpx
import asyncio

class ObscuraClient:
    def __init__(self, api_key: str, pool_id: str, base_url: str = "https://api.obscura.finance"):
        self.api_key = api_key
        self.pool_id = pool_id
        self.base_url = base_url
        self.client = httpx.AsyncClient(
            headers={"X-API-Key": api_key, "Content-Type": "application/json"},
            timeout=30.0
        )
    
    async def submit_signal(
        self,
        symbol: str,
        side: str,
        quantity: float,
        order_type: str = "market",
        priority: str = "normal",
        **kwargs
    ) -> dict:
        payload = {
            "symbol": symbol,
            "side": side,
            "quantity": quantity,
            "order_type": order_type,
            "priority": priority,
            **kwargs
        }
        
        response = await self.client.post(
            f"{self.base_url}/v2/pools/{self.pool_id}/signals",
            json=payload
        )
        response.raise_for_status()
        return response.json()
    
    async def get_signal_status(self, signal_id: str) -> dict:
        response = await self.client.get(
            f"{self.base_url}/v2/pools/{self.pool_id}/signals/{signal_id}"
        )
        response.raise_for_status()
        return response.json()

# Usage
async def main():
    client = ObscuraClient(
        api_key="obsc_live_...",
        pool_id="your-pool-uuid"
    )
    
    # Submit a signal
    result = await client.submit_signal(
        symbol="BTCUSDT",
        side="buy",
        quantity=0.1,
        leverage=10,
        priority="high",
        external_trade_id="my-bot-trade-001"
    )
    
    print(f"Signal queued: {result['signal_id']}")
    print(f"Subscribers: {result['estimated_subscribers']}")

asyncio.run(main())
```

### JavaScript/TypeScript

```typescript
import axios from 'axios';

class ObscuraClient {
  private apiKey: string;
  private poolId: string;
  private baseUrl: string;

  constructor(apiKey: string, poolId: string, baseUrl = 'https://api.obscura.finance') {
    this.apiKey = apiKey;
    this.poolId = poolId;
    this.baseUrl = baseUrl;
  }

  async submitSignal(signal: {
    symbol: string;
    side: 'buy' | 'sell';
    quantity: number;
    orderType?: 'market' | 'limit';
    price?: number;
    priority?: 'low' | 'normal' | 'high' | 'critical';
    externalTradeId?: string;
    [key: string]: any;
  }): Promise<any> {
    const response = await axios.post(
      `${this.baseUrl}/v2/pools/${this.poolId}/signals`,
      {
        symbol: signal.symbol,
        side: signal.side,
        quantity: signal.quantity,
        order_type: signal.orderType || 'market',
        priority: signal.priority || 'normal',
        external_trade_id: signal.externalTradeId,
        ...signal,
      },
      {
        headers: {
          'X-API-Key': this.apiKey,
          'Content-Type': 'application/json',
        },
      }
    );
    return response.data;
  }

  async getSignalStatus(signalId: string): Promise<any> {
    const response = await axios.get(
      `${this.baseUrl}/v2/pools/${this.poolId}/signals/${signalId}`,
      {
        headers: { 'X-API-Key': this.apiKey },
      }
    );
    return response.data;
  }
}

// Usage
const client = new ObscuraClient('obsc_live_...', 'your-pool-uuid');

const result = await client.submitSignal({
  symbol: 'BTCUSDT',
  side: 'buy',
  quantity: 0.1,
  leverage: 10,
  priority: 'high',
  externalTradeId: 'my-bot-trade-001',
});

console.log(`Signal queued: ${result.signal_id}`);
```

---

## 📞 Support

### Technical Support
- **Email**: b2b@obscura.finance
- **Discord**: #b2b-integration channel
- **Response Time**: < 4 hours (business days)

### Onboarding
- **Email**: partnerships@obscura.finance
- **Schedule a Call**: calendly.com/obscura/b2b

### API Status
- **Status Page**: status.obscura.finance
- **Uptime SLA**: 99.9%

---

## 📚 Related Documentation

- [System Architecture](./04_SYSTEM_ARCHITECTURE.md)
- [Security Model](./05_SECURITY_MODEL.md)
- [TEE Implementation](./02_TEE_IMPLEMENTATION.md)
- [API Gateway README](/api-gateway/README.md)
- [Conductor README](/obscura-conductor/README.md)

---

*Last Updated: December 2025*
