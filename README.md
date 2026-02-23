# Obscura Technical Documentation

## 🏗️ Building the Verifiable Reputation Layer for DeFi

Welcome to Obscura's technical documentation. This documentation explains how we've implemented **privacy-preserving copy trading** using Trusted Execution Environments (TEE), Zero-Knowledge Proofs (ZK), and decentralized architecture.

---

## 📚 Documentation Structure

### [01. Privacy Architecture Overview](./01_PRIVACY_ARCHITECTURE_OVERVIEW.md)
**Start here** to understand Obscura's privacy model and how we solve the privacy-reputation paradox.

**Topics Covered**:
- The privacy problem in copy trading
- Multi-layered privacy architecture
- Trust model and guarantees
- Compliance & standards
- Key differentiators

**Recommended for**: All audiences

---

### [02. TEE Implementation](./02_TEE_IMPLEMENTATION.md)
Deep dive into our **Trusted Execution Environment** implementation using Nillion nilCC with AMD SEV-SNP hardware attestation.

**Topics Covered**:
- What is a TEE and why it matters
- Citadel service architecture
- AMD SEV-SNP hardware attestation (via Nillion nilCC)
- Encryption scheme (AES-256-GCM with versioned keys)
- Hardware attestation
- API endpoints and integration
- Performance metrics
- Security guarantees

**Recommended for**: Security engineers, DevOps, Backend developers

**Key Technologies**: Nillion nilCC, AMD SEV-SNP, AES-256-GCM, Versioned Key Rotation

---

### [03. ZK Implementation](./03_ZK_IMPLEMENTATION.md)
Complete guide to our **Zero-Knowledge Proof** system for verifiable trading performance.

**Topics Covered**:
- What is a zero-knowledge proof
- Alchemist service architecture
- Noir circuit design and logic
- BN254 curve and Poseidon hash
- Honk proof system
- Proof generation and verification flow
- On-chain integration (Horizen L3)
- Privacy guarantees

**Recommended for**: Cryptography engineers, Blockchain developers, Researchers

**Key Technologies**: Noir, Barretenberg, BN254 curve, PLONK/Honk, Solidity

---

### [04. System Architecture](./04_SYSTEM_ARCHITECTURE.md)
High-level overview of Obscura's distributed microservices architecture.

**Topics Covered**:
- Core services (API Gateway, Sentinel, Citadel, Conductor, Alchemist)
- Data layer (PostgreSQL, Redis, InfluxDB)
- Data flow diagrams
- Security architecture
- Performance metrics
- Scalability and deployment

**Recommended for**: System architects, DevOps, Full-stack developers

**Key Technologies**: FastAPI, PostgreSQL, Redis, Docker, Kubernetes

---

### [05. Security Model & Threat Analysis](./05_SECURITY_MODEL.md)
Comprehensive security model with threat analysis and mitigation strategies.

**Topics Covered**:
- Threat model and assets
- Multi-layer defense (network, application, data, hardware)
- Attack scenarios and mitigations
- Zero-knowledge privacy guarantees
- Compliance (GDPR, SOC 2)
- Incident response plan

**Recommended for**: Security auditors, CISOs, Risk managers

---

### [06. Protocol Whitepaper & Decentralization Roadmap](./06_PROTOCOL_WHITEPAPER.md)
Academic whitepaper summary with detailed path to full decentralization.

**Topics Covered**:
- Sentinel DON (Decentralized Oracle Network) design
- BFT consensus mechanism (OCR protocol)
- Citadel MPC network (threshold cryptography)
- Alchemist prover marketplace
- Cryptoeconomic security model
- Phase-by-phase decentralization roadmap (2025-2027)
- Economic model and token utility

**Recommended for**: Researchers, Token economists, Node operators, Investors

**Key Technologies**: BFT consensus, Multi-Party Computation, Threshold signatures, Cryptoeconomics

---

### [07. B2B Integration Guide](./07_B2B_INTEGRATION_GUIDE.md)
Complete guide for **B2B platforms** (trading bots, signal providers, copy trading aggregators) to integrate with Obscura.

**Topics Covered**:
- Tenant onboarding and API key setup
- Creating and managing trading pools
- Signal submission (REST API, Batch, High-Volume Stream)
- Subscription modes (Fixed Amount, Advanced, Martingale)
- SDK examples (Python, JavaScript)
- Error handling and idempotency

**Recommended for**: B2B integration engineers, Trading bot developers, Platform architects

**Key Technologies**: REST API, Redis Streams, BullMQ-style queues, WebSocket

---

### [08. Supported Exchanges](./08_SUPPORTED_EXCHANGES.md)
Complete catalog of **100+ supported exchanges** via CCXT.

**Topics Covered**:
- Certified CEX exchanges (Binance, Bybit, OKX, etc.)
- Certified DEX exchanges (Hyperliquid, dYdX, Paradex)
- Exchange capabilities and features
- API setup instructions
- Testnet support
- Fee comparison

**Recommended for**: Traders, Integration engineers, DevOps

**Key Technologies**: CCXT, Exchange APIs, WebSocket

---

## 🔐 Privacy Technologies Explained

### Trusted Execution Environments (TEE)

**Problem**: How do we protect exchange API keys from everyone, including ourselves?

**Solution**: Hardware-isolated secure enclaves (AWS Nitro) where credentials are encrypted/decrypted. Even with root access to the server, plaintext keys cannot be extracted.

**Implementation**: Citadel service + Evervault + AWS Nitro Enclaves

**Read more**: [02_TEE_IMPLEMENTATION.md](./02_TEE_IMPLEMENTATION.md)

---

### Zero-Knowledge Proofs (ZK)

**Problem**: How can traders prove profitability without revealing their trades?

**Solution**: Cryptographic proofs that verify aggregate performance (total PnL, win rate) without exposing individual trade details.

**Implementation**: Alchemist service + Noir circuits + Horizen L3

**Read more**: [03_ZK_IMPLEMENTATION.md](./03_ZK_IMPLEMENTATION.md)

---

### Decentralized Architecture (Future)

**Problem**: How do we eliminate single points of trust?

**Solution**: Decentralized Oracle Network (DON) for data ingestion, distributed ZK prover marketplace.

**Status**: Roadmap Q2 2026 (Sentinel DON), Q4 2026 (Alchemist Network)

**Read more**: [01_PRIVACY_ARCHITECTURE_OVERVIEW.md](./01_PRIVACY_ARCHITECTURE_OVERVIEW.md)

---

## 🚀 Quick Start Guides

### For Developers

**Prerequisites**:
- Python 3.11+
- Node.js 18+
- PostgreSQL 15+
- Redis 7+
- Docker (recommended)

**Setup**:
```bash
# Clone repository
git clone https://github.com/obscura/obscura-platform
cd obscura-platform

# Start all services
docker-compose up -d

# Verify health
curl http://localhost:8000/health
```

**Service Ports**:
- API Gateway: 8000
- Sentinel: 8000
- Conductor: 8001
- Alchemist: 8003
- Citadel: 8004

**Next Steps**:
1. Read [System Architecture](./04_SYSTEM_ARCHITECTURE.md)
2. Review [Protocol Whitepaper & Decentralization Roadmap](./06_PROTOCOL_WHITEPAPER.md)
3. Explore individual service READMEs:
   - [API Gateway](/api-gateway/README.md)
   - [Sentinel](/sentinel-lite/README.md)
   - [Conductor](/obscura-conductor/README.md)
   - [Citadel](/citadel/README.md)
   - [Alchemist](/alchemist/README.md)

---

### For Security Auditors

**Focus Areas**:
1. **Credential Protection**: Review [TEE Implementation](./02_TEE_IMPLEMENTATION.md)
2. **Attack Scenarios**: Review [Security Model](./05_SECURITY_MODEL.md)
3. **ZK Soundness**: Review [ZK Implementation](./03_ZK_IMPLEMENTATION.md)
4. **Network Security**: Review [System Architecture](./04_SYSTEM_ARCHITECTURE.md)

**Verification Steps**:
```bash
# 1. Verify Citadel attestation
curl http://localhost:8004/attest

# 2. Verify ZK proof on-chain
npx hardhat verify --network horizen <contract_address>

# 3. Run security tests
npm run test:security

# 4. Check audit logs
docker exec postgres psql -c "SELECT * FROM audit_log WHERE action = 'decrypt_credentials';"
```

**Bug Bounty**: security@obscura.finance

---

### For Researchers

**Research Papers**:
- **Whitepaper**: [View on Google Drive](https://drive.google.com/file/d/11idbG1itLzy922oIIy_d9BXlnVlgQEAY/view)
- **Announcement Article**: [Read on Medium](https://medium.com/@obscuraprotocol/announcing-obscura-building-the-verifiable-reputation-layer-for-defi-741de78578d8)

**Key Innovations**:
1. **Hybrid Privacy**: Combining TEE + ZK for credential protection + performance verification
2. **Cross-Exchange Reputation**: Unified verifiable reputation across CEXs and DEXs
3. **Hardware-Backed Security**: AWS Nitro Enclaves for credential isolation
4. **On-Chain Verification**: ZK proofs verified and stored on Horizen L3

**Open Problems**:
- Recursive ZK proof aggregation for multi-period reports
- Decentralized prover marketplace incentive design
- Byzantine fault tolerant consensus for oracle network

---

## 📊 System Metrics

### Performance

| Metric | Current | Target (6mo) |
|--------|---------|--------------|
| Trade Ingestion Latency | <100ms | <50ms |
| Copy Trade Execution | 200-500ms | <200ms |
| ZK Proof Generation | 2-5s | <1s |
| Concurrent Users | 10,000 | 100,000 |
| Throughput | 100 exec/s | 1,000 exec/s |

### Security

| Metric | Value |
|--------|-------|
| Encryption | AES-256-GCM |
| ZK Security | ~128-bit (BN254) |
| Attestation | AWS Nitro Hardware |
| Uptime SLA | 99.9% |
| Incident Response | <1 hour |

---

## 🗺️ Roadmap

### Phase 1: Citadel MVP (Q4 2025) ✅ **COMPLETE**
- ✅ Centralized Sentinel for trade ingestion
- ✅ Citadel TEE for credential protection
- ✅ Alchemist ZK proof generation
- ✅ On-chain verification (Horizen L3)
- ✅ Basic copy trading functionality

### Phase 1.5: V2 Trading Platform (Q4 2025) ✅ **COMPLETE**
- ✅ **Trading Pools**: Curated, Custom, Partner pool types
- ✅ **B2B Multi-Tenancy**: Tenant isolation, API keys, scopes
- ✅ **Subscription Modes**: Fixed Amount (beginner) & Advanced (pro)
- ✅ **Martingale System**: Optional loss-recovery strategy
- ✅ **100+ Exchange Support**: CEX & DEX via CCXT
- ✅ **B2B Signal Submission**: REST API, Batch, High-Volume Streams
- ✅ **Priority Queuing**: BullMQ-style with dead-letter queues
- ✅ **Risk Management**: Daily limits, per-pair config, SL/TP

### Phase 2: Sentinel DON (Q2 2026)
- 🔄 Decentralized Oracle Network (DON)
- 🔄 Byzantine Fault Tolerant consensus
- 🔄 $ZEN token staking for node operators
- 🔄 Trustless data aggregation

### Phase 3: Alchemist Network (Q4 2026)
- 🔄 Decentralized ZK prover marketplace
- 🔄 Competitive proof generation
- 🔄 Recursive proof aggregation
- 🔄 Cross-chain reputation

**Legend**: ✅ Complete | 🔄 In Progress | ⏳ Planned

---

## 🔗 External Resources

### Official Links
- **Website**: https://obscura.finance (Coming Soon)
- **Twitter/X**: [@UseObscura](https://x.com/UseObscura)
- **GitHub**: https://github.com/obscura (Coming Soon)
- **Discord**: https://discord.gg/obscura (Coming Soon)

### Technology Partners
- **Horizen**: https://www.horizen.io/ (Privacy-focused blockchain)
- **Evervault**: https://evervault.com/ (Managed TEE infrastructure)
- **AWS Nitro**: https://aws.amazon.com/ec2/nitro/ (Hardware enclaves)
- **Noir**: https://noir-lang.org/ (ZK circuit DSL)

### Academic References
- **PLONK**: https://eprint.iacr.org/2019/953 (ZK proof system)
- **BN254 Curve**: https://eprint.iacr.org/2010/429 (Pairing-friendly curve)
- **Poseidon Hash**: https://eprint.iacr.org/2019/458 (ZK-optimized hash)
- **TEE Survey**: https://arxiv.org/abs/1904.06441 (Trusted execution)

---

## 📧 Contact

### For Technical Questions
- **Email**: dev@obscura.finance
- **Discord**: #dev-support channel

### For Security Issues
- **Email**: security@obscura.finance
- **PGP Key**: Available on request
- **Bug Bounty**: Up to $50,000 for critical vulnerabilities

### For Partnerships
- **Email**: partnerships@obscura.finance
- **Twitter DM**: [@UseObscura](https://x.com/UseObscura)

---

## 📄 License

This documentation is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

Source code licensed under [MIT License](../LICENSE) (where applicable).

---

## 🙏 Acknowledgments

Obscura is proudly backed by:
- **Horizen Genesis Program**: Infrastructure and ecosystem support
- **Community Contributors**: Open-source developers and researchers
- **Security Auditors**: Trail of Bits, Quantstamp (upcoming audits)

---

**Built with ❤️ by the Obscura team. Powered by privacy.**

*Last Updated: December 2025*
