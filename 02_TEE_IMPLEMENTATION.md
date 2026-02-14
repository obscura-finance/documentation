# Trusted Execution Environment (TEE) Implementation

## Overview

Obscura's **Citadel service** implements hardware-based Trusted Execution Environments (TEEs) using **AWS Nitro Enclaves** to provide the strongest possible protection for user exchange API credentials. This ensures that sensitive cryptographic keys are **never exposed in plaintext** outside of hardware-isolated memory.

## What is a Trusted Execution Environment?

A TEE is a **hardware-isolated secure area** within a processor that guarantees:

1. **Confidentiality**: Code and data are protected from unauthorized access
2. **Integrity**: Execution cannot be tampered with or observed
3. **Attestation**: Cryptographic proof of what code is running and where

### AWS Nitro Enclaves

AWS Nitro Enclaves provide:

- **Hardware-enforced isolation** from the parent EC2 instance
- **No persistent storage** - data only exists in enclave memory
- **No network access** - communication only via secure VSOCK channel
- **Cryptographic attestation** - signed PCRs (Platform Configuration Registers)
- **Memory encryption** - All enclave memory is encrypted at the hardware level

## Citadel Architecture

### v2.0: Evervault-Managed TEE (Current)

```
┌────────────────────────────────────────────────────────────────┐
│                     Citadel Service (FastAPI)                   │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                Evervault SDK Client                       │ │
│  │  • encrypt(plaintext) → "ev:ENCRYPTED:STRING..."          │ │
│  │  • decrypt(ciphertext) → plaintext                        │ │
│  └──────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              │ HTTPS + TLS 1.3                  │
│                              ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │          Evervault Managed Infrastructure                 │ │
│  │  ┌────────────────────────────────────────────────────┐  │ │
│  │  │        AWS Nitro Enclave (Managed by Evervault)    │  │ │
│  │  │  • Hardware-isolated cryptographic operations      │  │ │
│  │  │  • AES-256-GCM encryption                          │  │ │
│  │  │  • Automatic key rotation                          │  │ │
│  │  │  • Hardware attestation                            │  │ │
│  │  │  • Zero plaintext logging                          │  │ │
│  │  └────────────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

**Key Benefits**:

- **80% less code**: Simplified from 3,000 to 600 lines
- **10x simpler deployment**: 5 minutes vs 2-3 hours
- **Same security**: Still uses AWS Nitro Enclaves
- **Managed infrastructure**: No Terraform, no KMS setup
- **Deploy anywhere**: Not limited to AWS EC2

### v1.0: Custom Nitro Enclave (Deprecated)

```
┌────────────────────────────────────────────────────────────────┐
│                 EC2 Instance (Parent)                           │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │           Citadel Parent Service (FastAPI)                │ │
│  │  • Enclave Manager (Load Balancer)                        │ │
│  │  • Health Monitor                                         │ │
│  │  • Circuit Breaker                                        │ │
│  └──────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              │ VSOCK (CID:Port)                 │
│                              ▼                                  │
│  ┌───────────┬───────────┬───────────┬───────────┐            │
│  │ Enclave 1 │ Enclave 2 │ Enclave 3 │ ... (8x)  │            │
│  │ 2 vCPU    │ 2 vCPU    │ 2 vCPU    │ 2 vCPU    │            │
│  │ 4GB RAM   │ 4GB RAM   │ 4GB RAM   │ 4GB RAM   │            │
│  │           │           │           │           │            │
│  │ • Encrypt │ • Encrypt │ • Encrypt │ • Encrypt │            │
│  │ • Decrypt │ • Decrypt │ • Decrypt │ • Decrypt │            │
│  │ • Sign    │ • Sign    │ • Sign    │ • Sign    │            │
│  │ • Attest  │ • Attest  │ • Attest  │ • Attest  │            │
│  └───────────┴───────────┴───────────┴───────────┘            │
│                                                                 │
│  Performance: ~1,000 requests/sec (8 enclaves × 125 req/s)    │
└────────────────────────────────────────────────────────────────┘
```

**Note**: v1.0 has been completely removed in favor of the simpler Evervault approach.

## Security Model

### Encryption Scheme

**Algorithm**: AES-256-GCM (Authenticated Encryption)

- **Key Size**: 256 bits
- **Mode**: Galois/Counter Mode (provides both confidentiality and authenticity)
- **Authentication Tag**: 128 bits (prevents tampering)
- **Nonce**: Randomly generated per encryption

### Data Flow

#### Encryption Flow

```
User API Key (Plaintext)
    │
    ▼
[API Gateway] ──HTTP POST──▶ [Citadel]
    │                             │
    │                             ▼
    │                        Evervault.encrypt()
    │                             │
    │                             ▼
    │                    [AWS Nitro Enclave]
    │                    • Generate nonce
    │                    • Encrypt with AES-256-GCM
    │                    • Compute auth tag
    │                             │
    │                             ▼
    │◀────────────────────  "ev:ENCRYPTED..."
    │
    ▼
[PostgreSQL] ← Encrypted Ciphertext Stored
```

#### Decryption Flow

```
[Conductor] needs to execute trade
    │
    ▼
Fetch encrypted credentials from [PostgreSQL]
    │
    ▼
[Conductor] ──HTTP POST──▶ [Citadel]
    │                             │
    │                             ▼
    │                        Evervault.decrypt()
    │                             │
    │                             ▼
    │                    [AWS Nitro Enclave]
    │                    • Verify auth tag
    │                    • Decrypt with AES-256-GCM
    │                    • Return plaintext
    │                             │
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

**Evervault Managed** (Current):

- Keys stored in Evervault's infrastructure
- Automatic key rotation
- Hardware Security Module (HSM) backed
- FIPS 140-2 Level 3 compliant

**AWS KMS** (v1.0 - Deprecated):

- Customer Master Keys (CMKs) in AWS KMS
- Data Encryption Keys (DEKs) generated per enclave
- Envelope encryption scheme
- Key rotation every 90 days

## Hardware Attestation

### What is Attestation?

Attestation is **cryptographic proof** that:

1. The code running in the enclave is exactly what you expect
2. The enclave is running on genuine AWS Nitro hardware
3. The enclave has not been tampered with

### Attestation Document Components

```json
{
  "module_id": "i-1234567890abcdef0-enc0123456789abcd",
  "timestamp": 1730000000000,
  "digest": "SHA384",
  "pcrs": {
    "0": "000000...",  // Enclave image hash
    "1": "000000...",  // Kernel/init hash
    "2": "000000...",  // Application hash
    "8": "000000..."   // Certificate authority
  },
  "certificate": "-----BEGIN CERTIFICATE-----...",
  "cabundle": ["-----BEGIN CERTIFICATE-----..."],
  "public_key": "-----BEGIN PUBLIC KEY-----...",
  "user_data": null,
  "nonce": null
}
```

### PCR (Platform Configuration Register) Values

| PCR  | Description   | Purpose                       |
| ---- | ------------- | ----------------------------- |
| PCR0 | Enclave Image | Verifies exact enclave binary |
| PCR1 | Kernel & Init | Verifies boot integrity       |
| PCR2 | Application   | Verifies application code     |
| PCR8 | Certificate   | Verifies signing authority    |

**Verification Process**:

1. Enclave generates attestation document
2. AWS Nitro Root signs the document
3. Client verifies signature chain up to AWS Root CA
4. Client verifies PCR values match expected hashes
5. If all checks pass, client trusts the enclave

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
  "encrypted_data": "ev:RFVC:TTLHY04oVJlBuvPx:A7MeNriTKFES0Djl9uKaPvHCAn9PjSfOHu7tswXFCHF9:$"
}
```

### Decrypt Endpoint

```http
POST /decrypt
Content-Type: application/json
X-Internal-API-Key: <secret>

{
  "encrypted_data": "ev:RFVC:..."
}
```

**Response**:

```json
{
  "plaintext": "my_binance_api_key"
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
  "service": "citadel",
  "version": "2.0.0",
  "provider": "evervault"
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
- Network-level isolation via VPC
- No public internet exposure

### Access Control

| Service               | Encrypt | Decrypt | Reason                                     |
| --------------------- | ------- | ------- | ------------------------------------------ |
| **Sentinel**    | ✅      | ❌      | Needs to encrypt new credentials           |
| **Conductor**   | ❌      | ✅      | Needs to decrypt for trade execution       |
| **API Gateway** | ❌      | ❌      | Should never handle credentials directly   |
| **Alchemist**   | ❌      | ❌      | Only processes trade data, not credentials |

### Audit Logging

All operations logged:

```json
{
  "timestamp": "2025-01-15T16:30:00Z",
  "operation": "decrypt",
  "requestor": "conductor",
  "user_id": "masked_user_123",
  "success": true,
  "duration_ms": 15
}
```

**Sensitive data never logged**:

- ❌ Plaintext API keys
- ❌ Plaintext secrets
- ❌ Decrypted data
- ✅ Encrypted ciphertext (safe to log)
- ✅ Operation metadata
- ✅ Error messages (sanitized)

### Rate Limiting

- **Per service**: 100 requests/min
- **Per user**: 10 requests/min
- **Circuit breaker**: Trips after 5 consecutive failures
- **Backoff**: Exponential (1s, 2s, 4s, 8s, 16s)

## Performance Metrics

### v2.0 (Evervault)

| Metric          | Value      | Notes                 |
| --------------- | ---------- | --------------------- |
| Encrypt Latency | ~50ms      | Including network RTT |
| Decrypt Latency | ~50ms      | Including network RTT |
| Throughput      | 100+ req/s | Single instance       |
| Availability    | 99.9%      | Evervault SLA         |

### v1.0 (Custom Enclave)

| Metric          | Value       | Notes                       |
| --------------- | ----------- | --------------------------- |
| Encrypt Latency | ~5ms        | VSOCK local communication   |
| Decrypt Latency | ~5ms        | VSOCK local communication   |
| Throughput      | 1,000 req/s | 8 enclaves @ 125 req/s each |
| Availability    | 99.5%       | Self-managed                |

**Trade-off**: v2.0 is 10x simpler to deploy and manage, with minimal latency increase (~45ms) acceptable for non-HFT use cases.

## Compliance & Standards

### Regulatory Compliance

- **GDPR Article 32**: Technical and organizational measures for data security
- **SOC 2 Type II**: Security, availability, and confidentiality controls
- **CCPA**: Secure handling of personal information
- **PCI DSS**: Encryption of cardholder data (if applicable)

### Security Standards

- **FIPS 140-2 Level 3**: Hardware security module compliance (via AWS KMS/Evervault)
- **TLS 1.3**: All network communication encrypted
- **AES-256-GCM**: NIST-approved authenticated encryption
- **SHA-384**: Cryptographic hashing for attestation

### AWS Compliance

AWS Nitro Enclaves inherit EC2 compliance:

- ISO 27001
- ISO 27017
- ISO 27018
- FedRAMP
- HIPAA

## Deployment

### Evervault Setup (5 minutes)

```bash
# 1. Sign up at https://app.evervault.com/
# 2. Create a new App
# 3. Get App ID and API Key

# 4. Configure environment
cat > .env <<EOF
EVERVAULT_APP_ID=app_xxxxxxxxxxxxx
EVERVAULT_API_KEY=ev:key:xxxxxxxxxxxxx
CITADEL_INTERNAL_API_KEY=$(openssl rand -hex 32)
EOF

# 5. Run with Docker
docker-compose -f docker-compose.evervault.yml up -d

# 6. Test
curl http://localhost:8004/health
```

### Custom Enclave Setup (2-3 hours) - DEPRECATED

```bash
# Build enclave image
cd citadel/enclave
./build-eif.sh

# Deploy with Terraform
cd deployment/terraform
terraform init
terraform apply

# Configure parent service
export ENCLAVE_CID=16
export VSOCK_PORT=5000
```

## Threat Model

### Threats Mitigated

✅ **Database Breach**: Encrypted credentials useless without decryption key
✅ **Memory Dump**: Enclave memory is encrypted
✅ **Insider Threat**: No admin access to plaintext keys
✅ **Service Compromise**: Sentinel can't decrypt, Conductor can't re-encrypt
✅ **Side-Channel Attacks**: Hardware-enforced isolation

### Threats Not Mitigated

⚠️ **Compromised Enclave Code**: Malicious code inside enclave can access plaintext
⚠️ **AWS Infrastructure Compromise**: Trust in AWS Nitro hardware required
⚠️ **Time-of-Use Attack**: Plaintext exists in Conductor memory during execution

**Mitigations**:

- **Attestation**: Verify exact code running in enclave via PCRs
- **Code Review**: Open-source enclave code for transparency
- **Minimal Plaintext Lifetime**: Dispose keys immediately after use
- **Memory Clearing**: Explicit zeroing of sensitive memory

## Best Practices

### For Developers

1. **Never log plaintext**: Always mask sensitive data
2. **Minimize plaintext lifetime**: Decrypt, use, dispose immediately
3. **Verify attestation**: Check PCR values in production
4. **Rotate keys**: Force credential re-encryption periodically
5. **Monitor metrics**: Track decryption failures and latency spikes

### For Operators

1. **Isolate Citadel**: Deploy in private subnet, no public access
2. **Monitor health**: Alert on failed attestations or circuit breaker trips
3. **Backup encrypted data**: Database backups are safe (data is encrypted)
4. **Key rotation**: Rotate internal API key every 90 days
5. **Incident response**: Revoke compromised internal API keys immediately

## References

- **AWS Nitro Enclaves**: https://aws.amazon.com/ec2/nitro/nitro-enclaves/
- **Evervault Documentation**: https://docs.evervault.com/
- **NIST AES-GCM**: https://csrc.nist.gov/publications/detail/sp/800-38d/final
- **TEE Standards**: https://globalplatform.org/specs-library/
