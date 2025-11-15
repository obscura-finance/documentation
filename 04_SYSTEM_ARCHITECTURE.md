# System Architecture

## Overview

Obscura is a **distributed microservices architecture** designed for privacy-preserving copy trading. The system combines TEE security, ZK proofs, and real-time streaming for scalable, secure trading.

## Core Services

### 1. API Gateway (Port 8000)
- JWT authentication and authorization
- Rate limiting (100 req/min general, 10 req/min execution)
- CORS, input validation, circuit breakers
- Health check aggregation

### 2. Sentinel Lite (Port 8000)
- Real-time WebSocket trade ingestion (<100ms latency)
- Multi-exchange adapters (Binance, Coinbase)
- Trade normalization and Redis pub/sub
- Fallback REST polling

### 3. Citadel (Port 8004)
- TEE-secured credential encryption/decryption
- Evervault/AWS Nitro Enclaves
- AES-256-GCM encryption
- Internal API key authentication

### 4. Conductor (Port 8001)
- Copy trade execution orchestrator
- Risk management and position sizing
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
└────────────────────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                   API Gateway (8000)                    │
│  JWT Auth │ Rate Limiting │ CORS │ Load Balancing     │
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
- **In Use**: AWS Nitro Enclaves (TEE)

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
