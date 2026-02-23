# Trusted Execution Environment (TEE) Implementation

## Overview

Obscura's **Citadel service** implements hardware-based Trusted Execution Environments (TEEs) using **Nillion nilCC (Confidential Compute)** with **AMD SEV-SNP** hardware attestation to provide the strongest possible protection for user exchange API credentials. This ensures that sensitive cryptographic keys are **never exposed in plaintext** outside of hardware-isolated memory.

> **Migration History**: Citadel has evolved through three architectures:
> - **v1.0** (2025): Custom AWS Nitro Enclaves — complex, self-managed
> - **v2.0** (Late 2025): Evervault-managed TEE — simplified deployment
> - **v2.1** (Jan 2026, **Current**): Nillion nilCC — decentralized TEE with AMD SEV-SNP

## What is a Trusted Execution Environment?

A TEE is a **hardware-isolated secure area** within a processor that guarantees:

1. **Confidentiality**: Code and data are protected from unauthorized access
2. **Integrity**: Execution cannot be tampered with or observed
3. **Attestation**: Cryptographic proof of what code is running and where

### AMD SEV-SNP (Secure Encrypted Virtualization — Secure Nested Paging)

AMD SEV-SNP, used by Nillion nilCC, provides:

- **Hardware-enforced memory encryption** — All memory encrypted with per-VM keys at the hardware level
- **Integrity protection** — SNP adds memory integrity guarantees (prevents replay, remapping, and aliasing attacks)
- **Cryptographic attestation** — Hardware-signed attestation reports proving code identity and platform authenticity
- **No persistent state** — Secrets exist only in encrypted memory during execution
- **Strong isolation** — Even the hypervisor and host OS cannot access guest memory in plaintext

## Citadel Architecture

### v2.1: Nillion nilCC (Current)

```
┌────────────────────────────────────────────────────────────────┐
│                     Citadel Service (FastAPI v2.1.0)            │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │            VersionedCipher (encryption.py)                │ │
│  │  • encrypt(plaintext) → "nil:v{n}:{base64_payload}"      │ │
│  │  • decrypt(ciphertext) → plaintext                        │ │
│  │  • re_encrypt(payload) → migrate to latest key version    │ │
│  │  • Multi-version key support (v1, v2, ..., v9)            │ │
│  └──────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                   Running inside nilCC TEE                      │
│                              │                                  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │            Nillion nilCC Infrastructure                    │ │
│  │  ┌────────────────────────────────────────────────────┐  │ │
│  │  │        AMD SEV-SNP Hardware Enclave                │  │ │
│  │  │  • Hardware-level memory encryption                │  │ │
│  │  │  • AES-256-GCM authenticated encryption            │  │ │
│  │  │  • Versioned key rotation (nil:v{n}:payload)       │  │ │
│  │  │  • Hardware attestation via /nilcc/api/v2/report    │  │ │
│  │  │  • Zero plaintext logging                          │  │ │
│  │  └────────────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Key Benefits:                                                  │
│  • Decentralized infrastructure (not tied to single cloud)      │
│  • AMD SEV-SNP hardware attestation                             │
│  • Versioned key rotation without re-encrypting existing data   │
│  • Backward compatible with legacy payloads                     │
│  • Simple deployment via Docker                                 │
└────────────────────────────────────────────────────────────────┘
```

**Key Benefits over Previous Versions**:

- **Decentralized TEE**: Unlike Evervault (single vendor) or self-managed Nitro (single AWS account), Nillion provides decentralized confidential compute — aligning with Obscura's decentralization thesis
- **Hardware attestation**: AMD SEV-SNP provides equivalent (and in some aspects stronger) security guarantees to AWS Nitro Enclaves
- **Key versioning**: Built-in support for versioned encryption keys (`nil:v{n}:{payload}`) enabling seamless key rotation
- **Backward compatible**: Can decrypt legacy `nil:{payload}` and raw base64 formats from migration
- **Simple deployment**: Docker-based deployment with `docker-compose.nilcc.yml`

### Previous Versions (Deprecated)

#### v2.0: Evervault-Managed TEE (Deprecated)

Used Evervault SDK to proxy encryption through their managed AWS Nitro Enclaves. Encrypted data used `ev:ENCRYPTED:STRING...` format. **Fully replaced by nilCC in January 2026.**

#### v1.0: Custom AWS Nitro Enclave (Deprecated)

Self-managed enclave pool on EC2 with VSOCK communication, KMS integration, and Terraform deployment. Complex to operate (2-3 hours to deploy) and AWS-locked. **Removed in favor of Evervault, then nilCC.**

## Security Model

### Encryption Scheme

**Algorithm**: AES-256-GCM (Authenticated Encryption)

- **Key Size**: 256 bits (32 bytes, provided as 64 hex characters)
- **Mode**: Galois/Counter Mode (provides both confidentiality and authenticity)
- **Authentication Tag**: 128 bits (prevents tampering, appended by AESGCM)
- **Nonce**: 12 bytes, randomly generated per encryption operation (`os.urandom(12)`)

### Key Versioning

Citadel supports **multiple key versions** for seamless rotation:

```
Encryption Format:
  nil:v{version}:{base64_payload}

Examples:
  nil:v1:abc123...    (encrypted with version 1 key)
  nil:v2:def456...    (encrypted with version 2 key)
  nil:abc123...       (legacy format, treated as v1)

Key Configuration:
  CITADEL_MASTER_KEY      → Version 1 (primary)
  CITADEL_MASTER_KEY_V2   → Version 2 (rotation)
  CITADEL_MASTER_KEY_V3   → Version 3 (future)
  ... up to V9
```

**Key rotation behavior**:
- New encryptions always use the **highest available version**
- Decryption automatically detects the version from the prefix
- `re_encrypt()` method migrates old data to the current version
- `needs_migration()` checks if data uses an older key version

### Data Flow

#### Encryption Flow (Current — nilCC)

```
User API Key (Plaintext)
    │
    ▼
[API Gateway] ──HTTP POST──▶ [Citadel (nilCC TEE)]
    │                             │
    │                             ▼
    │                     VersionedCipher.encrypt()
    │                             │
    │                             ▼
    │                    [AMD SEV-SNP Enclave]
    │                    • Generate 12-byte nonce
    │                    • Encrypt with AES-256-GCM
    │                    • Compute auth tag
    │                    • Prefix with nil:v{n}:
    │                             │
    │                             ▼
    │◀────────────────────  "nil:v1:{base64_payload}"
    │
    ▼
[PostgreSQL] ← Encrypted Ciphertext Stored
```

#### Decryption Flow (Current — nilCC)

```
[Conductor] needs to execute trade
    │
    ▼
Fetch encrypted credentials from [PostgreSQL]
    │
    ▼
[Conductor] ──HTTP POST──▶ [Citadel (nilCC TEE)]
    │                             │
    │                             ▼
    │                     Parse "nil:v{n}:{payload}"
    │                     Select key for version n
    │                             │
    │                             ▼
    │                    [AMD SEV-SNP Enclave]
    │                    • Verify auth tag (tamper detection)
    │                    • Decrypt with AES-256-GCM
    │                    • Return plaintext
    │                             │
    │                             ▼
    │◀────────────────────  "plain_api_key"
    │
    ▼
Use plaintext key to sign order
(Key exists in memory only, never persisted)
    │
    ▼
Dispose of plaintext key after execution
```

### Key Management

**Nillion nilCC** (Current):

- Master keys configured via environment variables (`CITADEL_MASTER_KEY`)
- Keys stored securely within nilCC infrastructure
- Versioned key rotation without service downtime
- AMD SEV-SNP hardware-backed memory encryption
- Master key never leaves the TEE boundary in plaintext

**Legacy (Deprecated)**:

- **Evervault (v2.0)**: Keys in Evervault infrastructure, FIPS 140-2 Level 3 compliant HSMs
- **AWS KMS (v1.0)**: Customer Master Keys (CMKs), Data Encryption Keys (DEKs), envelope encryption

## Hardware Attestation

### What is Attestation?

Attestation is **cryptographic proof** that:

1. The code running in the TEE is exactly what you expect
2. The TEE is running on genuine AMD SEV-SNP hardware
3. The enclave has not been tampered with

### nilCC Attestation

Nillion nilCC provides attestation via a dedicated endpoint:

```
Attestation Endpoint: /nilcc/api/v2/report
TEE Type: AMD SEV-SNP
Provider: Nillion
```

The attestation report is served by the nilCC infrastructure itself (not by the Citadel application), ensuring that attestation is independent of the application code.

**Citadel exposes an info endpoint** at `/nilcc/attestation`:

```json
{
  "attestation_endpoint": "/nilcc/api/v2/report",
  "note": "Attestation report is provided by nilCC infrastructure",
  "tee_type": "AMD SEV-SNP",
  "provider": "nillion"
}
```

### AMD SEV-SNP Attestation vs AWS Nitro PCRs

| Feature | AMD SEV-SNP (Current) | AWS Nitro (Previous) |
|---------|----------------------|---------------------|
| **Attestation type** | Hardware-signed VM report | Signed enclave document |
| **Hardware root of trust** | AMD Secure Processor | AWS Nitro Security Chip |
| **Memory encryption** | Per-VM AES-128/256 keys | Per-enclave keys |
| **Integrity protection** | SNP: Reverse Map Table prevents remapping | VSOCK isolation |
| **Decentralized?** | Yes (Nillion network) | No (single AWS account) |
| **Endpoint** | `/nilcc/api/v2/report` | PCR values in attestation doc |

## API Endpoints

### Encrypt Endpoint

```http
POST /encrypt
Content-Type: application/json
X-Internal-API-Key: <secret>

{
  "plaintext": "my_binance_api_key"
}
```

**Response**:

```json
{
  "encrypted_data": "nil:v1:Kv3mHQ7x+2Np8a3W5ZqR1..."
}
```

### Decrypt Endpoint

```http
POST /decrypt
Content-Type: application/json
X-Internal-API-Key: <secret>

{
  "encrypted_data": "nil:v1:Kv3mHQ7x..."
}
```

**Response**:

```json
{
  "plaintext": "my_binance_api_key"
}
```

### Bulk Encrypt / Decrypt

```http
POST /encrypt-bulk
Content-Type: application/json
X-Internal-API-Key: <secret>

{
  "data": {
    "api_key": "my_api_key",
    "api_secret": "my_api_secret"
  }
}
```

**Response**:

```json
{
  "encrypted_data": {
    "api_key": "nil:v1:abc123...",
    "api_secret": "nil:v1:def456..."
  }
}
```

### Health Check

```http
GET /health
```

**Response**:

```json
{
  "status": "healthy",
  "service": "Citadel - Nillion nilCC Encryption Service",
  "version": "2.1.0",
  "tee": "nillion_nilcc",
  "encryption": {
    "mode": "nil_prefix",
    "available": true,
    "algorithm": "AES-256-GCM"
  }
}
```

### Attestation Info

```http
GET /nilcc/attestation
```

**Response**:

```json
{
  "attestation_endpoint": "/nilcc/api/v2/report",
  "note": "Attestation report is provided by nilCC infrastructure",
  "tee_type": "AMD SEV-SNP",
  "provider": "nillion"
}
```

## Security Features

### Authentication & Authorization

**Internal API Key**:

- All encrypt/decrypt endpoints require `X-Internal-API-Key` header
- Key generated with `openssl rand -hex 32`
- Stored in environment variables (not in code)
- Different key per environment (dev/staging/prod)

**Service-to-Service**:

- Only Sentinel and Conductor have access to Citadel
- Network-level isolation
- No public internet exposure

### Access Control

| Service               | Encrypt | Decrypt | Reason                                     |
| --------------------- | ------- | ------- | ------------------------------------------ |
| **Sentinel**    | ✅      | ❌      | Needs to encrypt new credentials           |
| **Conductor**   | ❌      | ✅      | Needs to decrypt for trade execution       |
| **API Gateway** | ❌      | ❌      | Should never handle credentials directly   |
| **Alchemist**   | ❌      | ❌      | Only processes trade data, not credentials |

### Evervault Data Rejection

Citadel v2.1 explicitly **rejects** legacy Evervault-encrypted data (`ev:` prefix):

```python
if request.encrypted_data.startswith("ev:"):
    raise HTTPException(
        status_code=400,
        detail="Evervault-encrypted data (ev:) not supported. Use nil: encrypted data."
    )
```

All existing credentials were migrated from Evervault to nilCC encryption via the `migrate_evervault_to_nilcc.py` script in January 2026.

### Audit Logging

All operations logged with structured logging (structlog):

```json
{
  "timestamp": "2026-02-23T16:30:00Z",
  "event": "encryption_successful",
  "plaintext_len": 64,
  "version": 1
}
```

**Sensitive data never logged**:

- ❌ Plaintext API keys
- ❌ Plaintext secrets
- ❌ Decrypted data
- ✅ Encrypted ciphertext prefix (safe to log)
- ✅ Operation metadata
- ✅ Error messages (sanitized)

### Rate Limiting

- **Per service**: 100 requests/min
- **Per user**: 10 requests/min
- **Circuit breaker**: Trips after 5 consecutive failures
- **Backoff**: Exponential (1s, 2s, 4s, 8s, 16s)

## Performance Metrics

### v2.1 (Nillion nilCC — Current)

| Metric          | Value      | Notes                 |
| --------------- | ---------- | --------------------- |
| Encrypt Latency | ~50ms      | Including network RTT |
| Decrypt Latency | ~50ms      | Including network RTT |
| Throughput      | 100+ req/s | Single instance       |
| Availability    | 99.9%      | nilCC infrastructure  |

### Previous Versions (For Reference)

| Version | Encrypt Latency | Throughput | Notes |
|---------|----------------|------------|-------|
| v2.0 (Evervault) | ~50ms | 100+ req/s | Managed, but centralized vendor |
| v1.0 (Custom Nitro) | ~5ms | 1,000 req/s | Fast but complex to operate |

## Deployment

### Nillion nilCC Setup (Current)

```bash
# 1. Configure environment
cat > .env <<EOF
CITADEL_MASTER_KEY=$(openssl rand -hex 32)
CITADEL_INTERNAL_API_KEY=$(openssl rand -hex 32)
ENVIRONMENT=production
APP_NAME=citadel
EOF

# 2. Run with Docker (nilCC mode)
docker-compose -f docker-compose.nilcc.yml up -d

# 3. Test
curl http://localhost:8004/health
```

**Docker Compose** (`docker-compose.nilcc.yml`):

```yaml
# Citadel runs inside Nillion nilCC TEE
# AMD SEV-SNP provides hardware-level memory encryption
services:
  citadel:
    build: .
    command: python -m uvicorn app.main_nilcc:app --host 0.0.0.0 --port 8004
    ports:
      - "8004:8004"
    environment:
      - CITADEL_MASTER_KEY=${CITADEL_MASTER_KEY}
      - CITADEL_INTERNAL_API_KEY=${CITADEL_INTERNAL_API_KEY}
```

## Compliance & Standards

### Regulatory Compliance

- **GDPR Article 32**: Technical and organizational measures for data security
- **SOC 2 Type II**: Security, availability, and confidentiality controls (preparation)
- **CCPA**: Secure handling of personal information
- **MiCA**: Crypto-Asset Service Provider requirements for credential protection

### Security Standards

- **AES-256-GCM**: NIST-approved authenticated encryption (SP 800-38D)
- **TLS 1.3**: All network communication encrypted
- **AMD SEV-SNP**: Hardware-backed memory encryption and attestation
- **SHA-384**: Cryptographic hashing for attestation

## Threat Model

### Threats Mitigated

✅ **Database Breach**: Encrypted credentials (`nil:v{n}:...`) useless without master key, which lives only in the TEE
✅ **Memory Dump**: AMD SEV-SNP encrypts all guest VM memory at the hardware level
✅ **Insider Threat**: No admin access to plaintext keys — master key inside hardware enclave
✅ **Service Compromise**: Sentinel can't decrypt, Conductor can't re-encrypt (access control)
✅ **Key Rotation**: Versioned encryption allows rotating keys without downtime
✅ **Vendor Lock-in**: Decentralized nilCC avoids dependence on any single cloud provider

### Threats Not Mitigated

⚠️ **Compromised TEE Code**: Malicious code inside the enclave can access plaintext
⚠️ **AMD Hardware Compromise**: Trust in AMD SEV-SNP hardware is required
⚠️ **Time-of-Use Attack**: Plaintext exists in Conductor memory during execution

**Mitigations**:

- **Attestation**: Verify exact code running via `/nilcc/api/v2/report`
- **Code Review**: Open-source application code for transparency
- **Minimal Plaintext Lifetime**: Dispose keys immediately after use
- **Memory Clearing**: Explicit zeroing of sensitive memory

## Best Practices

### For Developers

1. **Never log plaintext**: Always mask sensitive data
2. **Minimize plaintext lifetime**: Decrypt, use, dispose immediately
3. **Verify attestation**: Check attestation reports in production
4. **Rotate keys**: Use versioned keys (`CITADEL_MASTER_KEY_V2`, etc.)
5. **Monitor metrics**: Track decryption failures and latency spikes

### For Operators

1. **Isolate Citadel**: Deploy in private network, no public access
2. **Monitor health**: Alert on degraded health status or failed encryptions
3. **Backup encrypted data**: Database backups are safe (data is `nil:` encrypted)
4. **Key rotation**: Rotate master key by adding new version, then migrating data
5. **Incident response**: Revoke compromised internal API keys immediately

## References

- **Nillion nilCC**: https://docs.nillion.com/
- **AMD SEV-SNP**: https://www.amd.com/en/developer/sev.html
- **NIST AES-GCM**: https://csrc.nist.gov/publications/detail/sp/800-38d/final
- **TEE Standards**: https://globalplatform.org/specs-library/

---

*Last Updated: February 2026*
