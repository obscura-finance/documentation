# Obscura Privacy Architecture Overview

## Executive Summary

Obscura is a **privacy-preserving copy trading platform** that enables traders to build verifiable reputation while maintaining confidentiality of sensitive data. The platform synthesizes three core privacy-enhancing technologies:

1. **Trusted Execution Environments (TEE)** - Hardware-isolated credential management
2. **Zero-Knowledge Proofs (ZK)** - Verifiable performance without data exposure
3. **Decentralized Architecture** - Distributed trust model with no single point of failure

## The Privacy Problem

Traditional copy trading platforms force users to choose between **privacy** and **trust**:

- **To gain followers**: Traders must expose sensitive financial data (positions, balances, strategies)
- **To follow traders**: Users must trust unverifiable performance claims and screenshots
- **Platform risk**: Centralized credential storage creates honeypot for attackers

## Obscura's Solution: Multi-Layered Privacy Architecture

### Privacy Layer 1: Credential Protection (TEE)
**Technology**: AWS Nitro Enclaves via Evervault  
**Component**: Citadel Service  
**Protection**: Exchange API keys never exist in plaintext outside hardware enclaves

### Privacy Layer 2: Performance Verification (ZK)
**Technology**: ZK-SNARKs (Noir framework)  
**Component**: Alchemist Service  
**Protection**: Prove trading performance without revealing individual trades

### Privacy Layer 3: Data Isolation
**Technology**: Microservices architecture with encrypted data-at-rest  
**Components**: All services  
**Protection**: Service compromise does not cascade

### Privacy Layer 4: Decentralized Attestation (Future)
**Technology**: Decentralized Oracle Network (DON)  
**Component**: Sentinel DON (Roadmap Q2 2026)  
**Protection**: Byzantine Fault Tolerant consensus prevents data tampering

## Privacy Guarantees

### What Obscura Protects

✅ **Exchange API Credentials**: Never stored or transmitted in plaintext  
✅ **Individual Trade Details**: Not revealed in ZK proofs  
✅ **Portfolio Balances**: Only aggregate performance exposed  
✅ **Trading Strategies**: Derived from private trade patterns  
✅ **User Identities**: Pseudonymous on-chain reputation

### What Obscura Reveals

📊 **Aggregate Performance Metrics**: Total PnL, win rate, Sharpe ratio (via ZK proof)  
📊 **Copy Trade Signals**: When a followed trader makes a move  
📊 **On-Chain Reputation Score**: Cryptographically verified trading prowess

## Trust Model

### Current Phase (Centralized Bootstrap)
- **Trusted Components**: Citadel TEE, Alchemist ZK circuit
- **Trust Assumptions**: AWS Nitro Enclave hardware attestation, BN254 curve security
- **Verification**: Hardware attestation documents, on-chain proof verification

### Future Phase (Full Decentralization)
- **Sentinel DON**: Byzantine Fault Tolerant consensus (3f+1 nodes)
- **Alchemist Network**: Competitive marketplace for ZK provers
- **No Single Point of Trust**: Distributed verification

## Compliance & Standards

- **GDPR Ready**: Right to erasure, data portability, minimal data collection
- **SOC 2 Type II Path**: Audit-ready architecture
- **Zero-Knowledge Privacy**: Mathematical guarantee of privacy
- **Hardware-Backed Security**: FIPS 140-2 compliant AWS Nitro

## Architecture Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                    Privacy-Preserving Architecture              │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐   │
│  │   Sentinel  │─────▶│   Citadel   │─────▶│  Alchemist  │   │
│  │             │      │   (TEE)     │      │    (ZK)     │   │
│  │ Trade Data  │      │ Credentials │      │ Performance │   │
│  │ Ingestion   │      │ Protection  │      │  Proofs     │   │
│  └─────────────┘      └─────────────┘      └─────────────┘   │
│         │                     │                     │          │
│         │                     │                     │          │
│         ▼                     ▼                     ▼          │
│  ┌──────────────────────────────────────────────────────┐    │
│  │         Privacy-Preserving Data Layer                │    │
│  │  • Encrypted at rest (AES-256-GCM)                  │    │
│  │  • Encrypted in transit (TLS 1.3)                   │    │
│  │  • Encrypted in use (AWS Nitro Enclaves)            │    │
│  └──────────────────────────────────────────────────────┘    │
│         │                                                      │
│         ▼                                                      │
│  ┌──────────────────────────────────────────────────────┐    │
│  │           On-Chain Verification Layer                │    │
│  │  • Horizen L3 (Privacy-Focused Blockchain)          │    │
│  │  • HonkVerifier (ZK Proof Verification)              │    │
│  │  • Obscura Contract (Reputation Registry)            │    │
│  └──────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────┘
```

## Key Differentiators

### vs Traditional Copy Trading
- ❌ **Traditional**: Plaintext API keys in database, full trade history exposed
- ✅ **Obscura**: Hardware-encrypted credentials, ZK-verified performance

### vs Self-Hosted Solutions
- ❌ **Self-Hosted**: User manages infrastructure, no reputation portability
- ✅ **Obscura**: Managed privacy infrastructure, cross-exchange unified reputation

### vs Blockchain-Only Solutions
- ❌ **Blockchain-Only**: On-chain transparency exposes all trades
- ✅ **Obscura**: Off-chain privacy with on-chain verification

## Privacy Implementation Summary

| Component | Technology | Purpose | Security Level |
|-----------|-----------|---------|----------------|
| **Citadel** | AWS Nitro Enclaves (via Evervault) | Credential encryption/decryption | Hardware-backed |
| **Alchemist** | ZK-SNARKs (Noir/BN254) | Performance verification | Cryptographic |
| **Sentinel** | Encrypted database + Redis | Trade data ingestion | Application-level |
| **Conductor** | Secure key delegation to Citadel | Copy trade execution | Service-to-service |
| **API Gateway** | JWT + Rate limiting | Access control | Network-level |

## Next Steps

Explore detailed implementation guides:
- **[02_TEE_IMPLEMENTATION.md](./02_TEE_IMPLEMENTATION.md)** - Citadel & AWS Nitro Enclaves
- **[03_ZK_IMPLEMENTATION.md](./03_ZK_IMPLEMENTATION.md)** - Alchemist & ZK-SNARKs
- **[04_SYSTEM_ARCHITECTURE.md](./04_SYSTEM_ARCHITECTURE.md)** - Complete system design
- **[05_SECURITY_MODEL.md](./05_SECURITY_MODEL.md)** - Threat model & mitigations
