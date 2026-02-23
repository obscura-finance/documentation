# System Architecture

## Overview

Obscura is a **distributed microservices architecture** designed for privacy-preserving copy trading. The system combines TEE security, ZK proofs, and real-time streaming for scalable, secure trading.

## Core Services

### 1. API Gateway (Port 8000)
- JWT authentication and authorization
- B2B multi-tenant API key authentication
- Rate limiting (100 req/min general, 10 req/min execution)
- CORS, input validation, circuit breakers
- Health check aggregation
- V2 Pool management endpoints

### 2. Sentinel Lite (Port 8000)
- Real-time WebSocket trade ingestion (<100ms latency)
- Multi-exchange adapters (100+ exchanges via CCXT)
- Trade normalization and Redis pub/sub
- Fallback REST polling

### 3. Citadel (Port 8004)
- TEE-secured credential encryption/decryption
- Nillion nilCC / AMD SEV-SNP hardware attestation
- AES-256-GCM encryption with versioned keys (`nil:v{n}:{payload}`)
- Internal API key authentication

### 4. Conductor (Port 8001)
- Copy trade execution orchestrator
- B2B signal processing (REST, Batch, Stream)
- Pool-based fan-out execution
- Risk management and position sizing
- BullMQ-style priority queuing
- Dead-letter queue handling
- Citadel credential decryption
- Exchange API execution

### 5. Alchemist (Port 8003)
- ZK proof generation (Noir/Barretenberg)
- Trading performance verification
- On-chain proof submission (Horizen L3)
- Reputation registry

## Architecture Diagram

```
┌────────────────────────────────────────────────────────┐
│                     User/Client                        │
│         (D2C Web/App  |  B2B Trading Platforms)        │
└────────────────────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                   API Gateway (8000)                    │
│  JWT Auth │ API Keys │ Rate Limiting │ Multi-Tenant   │
└────────────────────────────────────────────────────────┘
          │              │              │
          ▼              ▼              ▼
┌────────────┐  ┌────────────┐  ┌────────────┐
│  Sentinel  │  │ Conductor  │  │ Alchemist  │
│   (8000)   │  │   (8001)   │  │   (8003)   │
│            │  │            │  │            │
│ WebSocket  │  │   Copy     │  │    ZK      │
│  Trading   │  │  Trading   │  │  Proofs    │
│  Data      │  │ Execution  │  │            │
│ 100+ CEX   │  │ B2B Pools  │  │            │
└────────────┘  └────────────┘  └────────────┘
      │              │                  │
      │              ▼                  │
      │         ┌────────────┐          │
      │         │  Citadel   │          │
      │         │   (8004)   │          │
      │         │    TEE     │          │
      │         └────────────┘          │
      │              │                  │
      ▼              ▼                  ▼
┌─────────────────────────────────────────────┐
│              Data Layer                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │PostgreSQL│  │  Redis   │  │ Horizen  │  │
│  │  (5432)  │  │  (6379)  │  │   L3     │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘
```

## Data Flow

### Copy Trade Flow
```
1. Trader executes trade on Binance
2. Sentinel receives WebSocket event (<100ms)
3. Store trade in PostgreSQL + publish to Redis
4. Conductor receives signal from Redis
5. Query followers, calculate position sizes
6. Decrypt credentials via Citadel
7. Execute orders on follower accounts
8. Log executions (total: 200-500ms)
```

### ZK Proof Flow
```
1. Trader accumulates trades (e.g., monthly)
2. Alchemist queries trade data from Sentinel
3. Transform data for Noir circuit
4. Generate ZK proof (2-5 seconds)
5. Submit to Horizen L3 blockchain
6. On-chain verification and storage
```

## Security Model

### Network Layers
- **Public Subnet**: API Gateway only (TLS 1.3, DDoS protection)
- **Private Subnet**: All services (no internet access)
- **Data Subnet**: Databases (VPC-only access)

### Authentication
- **Users**: JWT tokens (1-hour expiry)
- **Services**: Internal API keys
- **TEE**: Hardware attestation

### Encryption
- **At Rest**: AES-256 database encryption
- **In Transit**: TLS 1.3
- **In Use**: Nillion nilCC / AMD SEV-SNP (TEE)

## Performance Metrics

| Component | Latency | Throughput | Capacity |
|-----------|---------|------------|----------|
| API Gateway | 10-50ms | 10K req/s | 100K users |
| Sentinel | <100ms | 50 events/s | 10K connections |
| Conductor | 200-500ms | 100 exec/s | 1K concurrent |
| Citadel | ~50ms | 100 req/s | Managed |
| Alchemist | 2-5s | 100 proofs/hr | Queued |

## Scalability

- **Horizontal**: All services are stateless and can scale independently
- **Database**: Primary + 2 read replicas, partitioned by date
- **Redis**: Cluster mode (16,384 hash slots)
- **Load Balancing**: Weighted round-robin with health checks

## Deployment

See individual service READMEs for detailed deployment instructions:
- [API Gateway](/api-gateway/README.md)
- [Sentinel](/sentinel-lite/README.md)
- [Conductor](/obscura-conductor/README.md)
- [Citadel](/citadel/README.md)
- [Alchemist](/alchemist/README.md)

---

## V2 Features (December 2025)

### Multi-Tenant B2B Architecture

Obscura supports both D2C (direct-to-consumer) and B2B (business-to-business) clients with full tenant isolation.

```
┌─────────────────────────────────────────────────────────────┐
│                    Tenant Model                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Tenant Zero (D2C)        B2B Tenant A       B2B Tenant B   │
│  ┌─────────────┐         ┌─────────────┐    ┌───────────┐   │
│  │ Curated     │ ◄─────► │ Partner     │    │ Partner   │   │
│  │ Pools       │ Visible │ Pools       │    │ Pools     │   │
│  │ (Global)    │         │ (Isolated)  │    │ (Isolated)│   │
│  └─────────────┘         └─────────────┘    └───────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Trading Pools

Three pool types with different visibility:

| Pool Type | Created By | Visible To |
|-----------|------------|------------|
| **Curated** | Obscura admins | All users (cross-tenant) |
| **Custom** | D2C users | Creator only |
| **Partner** | B2B tenants | Tenant's users only |

### Subscription Modes

**Fixed Amount Mode** (Beginners):
- Plain-language risk parameters
- "How much are you ready to lose?"
- Automatic position sizing

**Advanced Mode** (Experienced):
- Per-pair configuration
- Custom SL/TP per symbol
- Martingale strategy support
- Daily loss limits

### B2B Signal Submission

Three integration options based on volume:

| Method | Volume | Latency | Use Case |
|--------|--------|---------|----------|
| REST API | <100/min | ~50ms | Standard integration |
| Batch API | <1000/batch | ~100ms | Portfolio rebalancing |
| Redis Stream | >100/min | ~10ms | High-frequency bots |

### Priority Queuing (BullMQ-style)

| Priority | Weight | Use Case |
|----------|--------|----------|
| `critical` | 0 | Emergency exits, liquidation prevention |
| `high` | 10 | Important signals |
| `normal` | 50 | Standard signals |
| `low` | 100 | Batch processing |

### Supported Exchanges

100+ exchanges via CCXT including:

**Certified CEX**: Binance, Bybit, OKX, Gate.io, KuCoin, Bitget, MEXC, HTX, BingX

**Certified DEX**: Hyperliquid, dYdX, Paradex, Apex Pro

See [08_SUPPORTED_EXCHANGES.md](./08_SUPPORTED_EXCHANGES.md) for the full list.

---

*Last Updated: February 2026*

- [Citadel](/citadel/README.md)
- [Alchemist](/alchemist/README.md)
