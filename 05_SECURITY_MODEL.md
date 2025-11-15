# Security Model & Threat Analysis

## Security Objectives

Obscura's security model is designed to protect:

1. **User Credentials**: Exchange API keys never exposed in plaintext
2. **Trading Data**: Individual trade details remain private
3. **System Integrity**: Services cannot be compromised to exfiltrate data
4. **Platform Availability**: DDoS and resource exhaustion attacks mitigated

## Threat Model

### Assets to Protect

| Asset | Sensitivity | Storage | Protection |
|-------|-------------|---------|------------|
| Exchange API Keys | **CRITICAL** | PostgreSQL (encrypted) | TEE (Citadel) |
| Individual Trades | **HIGH** | PostgreSQL | Access control, ZK proofs |
| User Balances | **HIGH** | PostgreSQL | Encrypted fields |
| JWT Tokens | **MEDIUM** | Redis cache | Short TTL, rotation |
| Trading Strategies | **HIGH** | Derived from trades | ZK proofs |

### Threat Actors

1. **External Attackers**: Internet-based attacks (DDoS, injection, etc.)
2. **Malicious Users**: Users attempting privilege escalation
3. **Insider Threats**: Compromised admin or developer accounts
4. **Service Compromise**: Exploited vulnerabilities in microservices
5. **Infrastructure Compromise**: Cloud provider or datacenter breach

## Multi-Layer Defense

### Layer 1: Network Security

**Threats Mitigated**:
- DDoS attacks
- Man-in-the-middle (MITM)
- Network sniffing
- Unauthorized access

**Mitigations**:
```
┌─────────────────────────────────────────────────┐
│ TLS 1.3 Encryption (All traffic)                │
│ • Perfect Forward Secrecy (PFS)                 │
│ • Certificate pinning                           │
│ • HSTS headers                                  │
├─────────────────────────────────────────────────┤
│ Web Application Firewall (WAF)                  │
│ • SQL injection prevention                      │
│ • XSS filtering                                 │
│ • Rate limiting (per IP)                        │
├─────────────────────────────────────────────────┤
│ DDoS Protection                                 │
│ • CloudFlare/AWS Shield                         │
│ • IP reputation filtering                       │
│ • Geo-blocking suspicious regions               │
├─────────────────────────────────────────────────┤
│ Network Segmentation                            │
│ • Public subnet: API Gateway only               │
│ • Private subnet: Internal services             │
│ • Data subnet: Databases (no internet)          │
└─────────────────────────────────────────────────┘
```

### Layer 2: Application Security

**Threats Mitigated**:
- Authentication bypass
- Authorization escalation
- Session hijacking
- API abuse

**Mitigations**:

**JWT Authentication**:
```python
# Token structure
{
  "user_id": "uuid",
  "role": "trader|follower|admin",
  "exp": 3600,  # 1 hour
  "iat": timestamp
}

# Token validation
def verify_token(token: str) -> User:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        if payload["exp"] < time.time():
            raise TokenExpired()
        return get_user(payload["user_id"])
    except Exception:
        raise Unauthorized()
```

**Rate Limiting** (Redis-backed):
```python
# Per-user limits
general_endpoints: 100 requests/minute
execution_endpoints: 10 requests/minute
auth_endpoints: 5 requests/minute

# Per-IP limits (prevent enumeration)
login_attempts: 5 per 15 minutes
registration: 3 per hour
```

**Input Validation**:
```python
# Pydantic models for all endpoints
class ConnectExchangeRequest(BaseModel):
    exchange: Literal["binance", "coinbase"]
    api_key: str = Field(min_length=32, max_length=128)
    api_secret: str = Field(min_length=32, max_length=128)
    
    @validator("api_key", "api_secret")
    def validate_credentials(cls, v):
        if not v.isprintable():
            raise ValueError("Invalid characters")
        return v
```

### Layer 3: Data Security

**Threats Mitigated**:
- Data theft (database breach)
- Insider access to sensitive data
- Data leakage in logs
- Backup exposure

**Mitigations**:

**Encryption at Rest** (PostgreSQL):
```sql
-- Enable encryption
ALTER SYSTEM SET ssl = on;
ALTER SYSTEM SET ssl_cert_file = '/path/to/cert.pem';

-- Encrypted columns
CREATE TABLE exchange_connections (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    exchange VARCHAR(50),
    encrypted_api_key TEXT,  -- Citadel encrypted
    encrypted_api_secret TEXT,  -- Citadel encrypted
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Field-level encryption for PII
CREATE EXTENSION pgcrypto;
INSERT INTO users (email) VALUES (
    pgp_sym_encrypt('user@example.com', 'encryption_key')
);
```

**Audit Logging**:
```json
{
  "timestamp": "2025-01-15T16:30:00Z",
  "service": "conductor",
  "action": "decrypt_credentials",
  "user_id": "masked_user_123",
  "resource": "exchange_connection:uuid",
  "success": true,
  "ip_address": "10.0.1.5",
  "user_agent": "obscura-conductor/1.0"
}
```

**Sensitive Data Masking**:
```python
# Never log plaintext credentials
logger.info(f"Decrypting credentials for user {user_id[:8]}...")  # First 8 chars only
logger.info(f"API key: {api_key[:4]}...{api_key[-4:]}")  # Mask middle
```

### Layer 4: Hardware Security (TEE)

**Threats Mitigated**:
- Memory dumps
- Service compromise leading to credential theft
- Insider access to plaintext keys
- Side-channel attacks

**Mitigations**:

**AWS Nitro Enclaves** (via Evervault):
```
Hardware Isolation:
├─ CPU cores allocated exclusively to enclave
├─ Memory encrypted at hardware level
├─ No persistent storage (ephemeral only)
├─ No network access (VSOCK only)
└─ Cryptographic attestation

Trust Model:
├─ Trust AWS Nitro hardware (SGX-like)
├─ Trust Evervault's enclave code
├─ Verify via attestation documents
└─ PCR values match expected hashes
```

**Attestation Verification**:
```typescript
async function verifyAttestati on(attestationDoc: Buffer): Promise<boolean> {
  // 1. Verify signature chain to AWS Root CA
  const isValidSignature = await verifySignatureChain(attestationDoc);
  
  // 2. Extract PCR values
  const pcrs = extractPCRs(attestationDoc);
  
  // 3. Verify PCR0 (enclave image) matches expected
  const expectedPCR0 = "3a5b9c2d1e...";  // Known good hash
  if (pcrs.PCR0 !== expectedPCR0) {
    throw new Error("Enclave image mismatch");
  }
  
  // 4. Verify timestamp is recent (prevent replay)
  const timestamp = extractTimestamp(attestationDoc);
  if (Date.now() - timestamp > 300000) {  // 5 minutes
    throw new Error("Attestation too old");
  }
  
  return isValidSignature && pcrs.PCR0 === expectedPCR0;
}
```

## Attack Scenarios & Mitigations

### Scenario 1: Database Breach

**Attack**: Attacker gains read access to PostgreSQL database

**What's Exposed**:
- ✅ Encrypted API keys (useless without Citadel)
- ✅ User emails (hashed)
- ✅ Trade data (but not credentials)

**What's Protected**:
- ❌ Plaintext API keys (encrypted by Citadel)
- ❌ Trading strategies (derived, not stored)

**Mitigation**:
- Encryption at rest (AES-256)
- Citadel-encrypted credentials
- Database access logs
- Regular security audits

**Impact**: **LOW** - No plaintext credentials exposed

### Scenario 2: Service Compromise (Conductor)

**Attack**: Attacker exploits vulnerability in Conductor service

**What's Exposed**:
- ✅ Temporarily decrypted credentials (in memory during execution)
- ✅ Follower list
- ✅ Execution logs

**What's Protected**:
- ❌ Stored credentials (encrypted, can't re-encrypt)
- ❌ Other users' credentials

**Mitigation**:
- Minimize plaintext lifetime (dispose after use)
- Memory clearing (explicit zeroing)
- Service isolation (can't access Sentinel data)
- Circuit breaker on Citadel (detect abuse)

**Impact**: **MEDIUM** - Temporary credential exposure for active executions only

### Scenario 3: Citadel TEE Compromise

**Attack**: Attacker compromises enclave code or AWS infrastructure

**What's Exposed**:
- ✅ All credentials (plaintext in enclave memory)

**What's Protected**:
- Nothing (full compromise)

**Mitigation**:
- Attestation verification (detect compromised code)
- Open-source enclave code (transparency)
- Regular security audits
- Bug bounty program
- Incident response plan (force key rotation)

**Impact**: **CRITICAL** - Requires immediate user credential rotation

**Likelihood**: **VERY LOW** - Requires breaking AWS Nitro hardware

### Scenario 4: Man-in-the-Middle (API Gateway → Services)

**Attack**: Attacker intercepts traffic between API Gateway and internal services

**What's Exposed**:
- ✅ Request/response data
- ✅ User IDs, trade signals

**What's Protected**:
- ❌ Credentials (still encrypted)
- ❌ JWT secrets (not transmitted)

**Mitigation**:
- VPC isolation (no external routes)
- mTLS between services (mutual authentication)
- Network encryption (WireGuard)
- Regular penetration testing

**Impact**: **LOW** - Limited value without credentials

### Scenario 5: Insider Threat (Malicious Admin)

**Attack**: Internal employee attempts to steal user credentials

**What Can Admin Do**:
- ✅ Access databases (encrypted data only)
- ✅ View logs (sanitized, no plaintext)
- ✅ Query Citadel (rate-limited, logged)

**What Can't Admin Do**:
- ❌ Decrypt credentials (no access to Citadel internal key)
- ❌ Extract plaintext from enclaves
- ❌ Modify attestation documents

**Mitigation**:
- Principle of least privilege
- Audit all Citadel requests
- Separation of duties (no single admin)
- Regular access reviews
- Anomaly detection (unusual Citadel usage)

**Impact**: **LOW** - Admin cannot access plaintext credentials

## Zero-Knowledge Privacy Guarantees

### What ZK Proofs Protect

**Hidden (Witness)**:
- Individual trade entry/exit prices
- Trade sizes (quantities)
- Exact timestamps
- Trading pairs
- Order of trades

**Revealed (Public Outputs)**:
- Total PnL
- Win rate (%)
- Sharpe ratio
- Total number of trades

### ZK Attack Resistance

**Proof Forgery**:
- Mitigation: BN254 curve (~128-bit security)
- Attack Cost: 2^128 operations (infeasible)

**Witness Extraction**:
- Mitigation: Zero-knowledge property (soundness + completeness)
- Attack: Cannot reverse engineer trades from proof

**Replay Attacks**:
- Mitigation: Nonce + timestamp in commitment
- Attack: Old proofs rejected on-chain

## Compliance & Auditing

### Audit Trail

All sensitive operations logged:
```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    service VARCHAR(50),
    action VARCHAR(100),
    user_id UUID,
    resource_type VARCHAR(50),
    resource_id UUID,
    success BOOLEAN,
    ip_address INET,
    error_message TEXT
);

-- Retention: 7 years (compliance)
-- Backup: Daily encrypted backups
-- Access: Read-only for security team
```

### GDPR Compliance

**Right to Erasure**:
```python
async def delete_user_data(user_id: UUID):
    # 1. Delete exchange connections (encrypted credentials)
    await db.execute("DELETE FROM exchange_connections WHERE user_id = ?", user_id)
    
    # 2. Anonymize trades (keep for analytics)
    await db.execute("UPDATE trades SET user_id = NULL WHERE user_id = ?", user_id)
    
    # 3. Delete user account
    await db.execute("DELETE FROM users WHERE id = ?", user_id)
    
    # 4. Invalidate JWT tokens
    await redis.delete(f"user_session:{user_id}")
    
    # Note: On-chain data (ZK proofs) is immutable, but pseudonymous
```

**Data Minimization**:
- Only collect necessary data (no email verification for trading)
- Short JWT expiry (1 hour)
- Automatic session cleanup (24 hours)

### SOC 2 Type II

**Security Controls**:
- Encryption (at rest, in transit, in use)
- Access control (RBAC)
- Audit logging (all operations)
- Vulnerability management (weekly scans)
- Incident response (24/7 on-call)

## Security Best Practices

### For Developers

1. **Never log sensitive data**: API keys, secrets, plaintext credentials
2. **Validate all inputs**: Use Pydantic models, sanitize SQL
3. **Use parameterized queries**: Prevent SQL injection
4. **Implement rate limiting**: Prevent brute force and DoS
5. **Follow least privilege**: Services only access what they need

### For Operators

1. **Rotate secrets regularly**: Internal API keys every 90 days
2. **Monitor anomalies**: Unusual Citadel usage, failed authentications
3. **Patch vulnerabilities**: Weekly dependency updates
4. **Backup encrypted**: Daily database backups with encryption
5. **Test disaster recovery**: Quarterly DR drills

### For Users

1. **Use API keys with read-only + trade permissions only**: Never grant withdrawal permissions
2. **Enable IP whitelisting**: Restrict API keys to known IPs
3. **Rotate keys periodically**: Every 6 months minimum
4. **Monitor activity**: Review execution logs regularly
5. **Report suspicious activity**: Contact security@obscura.network

## Incident Response Plan

### Severity Levels

| Level | Description | Response Time | Actions |
|-------|-------------|---------------|---------|
| P0 | Credential breach, service down | Immediate | Page on-call, isolate systems |
| P1 | Service degradation, failed attestation | 1 hour | Investigate, mitigate |
| P2 | Failed authentication spike | 4 hours | Monitor, alert users |
| P3 | Non-critical bug | 24 hours | Create ticket, schedule fix |

### Breach Response Steps

1. **Detect**: Automated monitoring triggers alert
2. **Contain**: Isolate compromised service, revoke access
3. **Investigate**: Analyze logs, determine scope
4. **Notify**: Inform affected users within 72 hours (GDPR)
5. **Remediate**: Patch vulnerability, force key rotation
6. **Review**: Post-mortem, update security controls

## Conclusion

Obscura's security model provides **defense in depth** with multiple layers of protection:

- **Network**: TLS 1.3, WAF, DDoS protection
- **Application**: JWT auth, rate limiting, input validation
- **Data**: Encryption at rest, audit logging, field-level encryption
- **Hardware**: TEE (AWS Nitro Enclaves), attestation, memory encryption
- **Cryptographic**: ZK proofs, BN254 curve, Poseidon hash

**Key Insight**: Even if multiple layers are breached, credentials remain protected by hardware-enforced TEE isolation.

## References

- **AWS Nitro Security**: https://aws.amazon.com/ec2/nitro/
- **Evervault Security**: https://docs.evervault.com/security
- **ZK Security**: https://eprint.iacr.org/2019/953 (PLONK)
- **OWASP Top 10**: https://owasp.org/www-project-top-ten/
- **NIST Cryptography**: https://csrc.nist.gov/
