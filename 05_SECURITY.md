# Security Analysis
## ElectricSQL Local-First Security

---

## 1. Security Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Layer 1: Network Security                                   │
│  ├── TLS 1.3 for all connections                            │
│  ├── Certificate pinning (mobile)                           │
│  └── WAF / DDoS protection                                  │
│                                                              │
│  Layer 2: Authentication                                     │
│  ├── JWT tokens with short expiry                           │
│  ├── Refresh token rotation                                 │
│  └── Device binding                                         │
│                                                              │
│  Layer 3: Authorization                                      │
│  ├── Role-based access control (RBAC)                       │
│  ├── Row-level security (RLS) in PostgreSQL                 │
│  └── Shape-level permissions in Electric                    │
│                                                              │
│  Layer 4: Data Protection                                    │
│  ├── Encryption at rest (SQLite, PostgreSQL)                │
│  ├── Encryption in transit (TLS)                            │
│  └── Client-side encryption (optional)                      │
│                                                              │
│  Layer 5: Audit & Monitoring                                 │
│  ├── All operations logged                                  │
│  ├── Anomaly detection                                      │
│  └── Compliance reporting                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Threat Model

### 2.1 STRIDE Analysis

| Threat | Risk | Mitigation |
|--------|------|------------|
| **S**poofing | Attacker impersonates user | JWT + device binding |
| **T**ampering | Data modification in transit | TLS + CRDT checksums |
| **R**epudiation | Deny sync operations | Audit logs + signed ops |
| **I**nformation Disclosure | Data leak | Encryption + RLS |
| **D**enial of Service | Overload sync service | Rate limiting + WAF |
| **E**levation of Privilege | Access other vendor data | RLS + tenant isolation |

### 2.2 Attack Vectors

```
┌─────────────────────────────────────────────────────────────┐
│                    ATTACK VECTORS                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. STOLEN DEVICE                                            │
│  ───────────────                                             │
│  Risk: Physical access to local SQLite DB                   │
│  Impact: Data exposure for synced barcodes                  │
│  Mitigation:                                                 │
│  • SQLite encryption (SQLCipher)                            │
│  • Remote wipe capability                                   │
│  • Auto-logout after inactivity                             │
│  • Biometric/PIN lock                                       │
│                                                              │
│  2. MAN-IN-THE-MIDDLE                                        │
│  ────────────────────                                        │
│  Risk: Intercept sync traffic                               │
│  Impact: Data exposure, CRDT injection                      │
│  Mitigation:                                                 │
│  • TLS 1.3 mandatory                                        │
│  • Certificate pinning                                      │
│  • HSTS headers                                             │
│                                                              │
│  3. MALICIOUS INSIDER                                        │
│  ────────────────────                                        │
│  Risk: Employee with valid credentials                      │
│  Impact: Data theft, sabotage                               │
│  Mitigation:                                                 │
│  • Least privilege access                                   │
│  • Audit logging                                            │
│  • Anomaly detection                                        │
│  • Data access reviews                                      │
│                                                              │
│  4. TOKEN THEFT                                              │
│  ────────────                                                │
│  Risk: JWT stolen from device/memory                        │
│  Impact: Unauthorized sync operations                       │
│  Mitigation:                                                 │
│  • Short token expiry (15 min)                              │
│  • Secure token storage (Keychain/Keystore)                 │
│  • Token binding to device ID                               │
│  • Refresh token rotation                                   │
│                                                              │
│  5. REPLAY ATTACK                                            │
│  ──────────────                                              │
│  Risk: Re-send old sync operations                          │
│  Impact: Data corruption, duplicate entries                 │
│  Mitigation:                                                 │
│  • CRDT timestamps prevent duplicates                       │
│  • Operation IDs (idempotency)                              │
│  • Nonce in requests                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Authentication Design

### 3.1 JWT Token Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    JWT TOKEN FLOW                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. LOGIN                                                    │
│  ────────                                                    │
│  Client ──► POST /auth/login ──► Auth Service               │
│             {username, password,                             │
│              device_id, device_fingerprint}                  │
│                                                              │
│  Auth Service:                                               │
│  • Validate credentials                                      │
│  • Generate access token (15 min TTL)                       │
│  • Generate refresh token (7 days TTL)                      │
│  • Bind tokens to device_id                                 │
│                                                              │
│  Response:                                                   │
│  {access_token, refresh_token, expires_in}                  │
│                                                              │
│  2. API CALLS                                                │
│  ──────────                                                  │
│  Client ──► Any API ──► Middleware validates JWT            │
│  Header: Authorization: Bearer <access_token>               │
│                                                              │
│  3. TOKEN REFRESH                                            │
│  ──────────────                                              │
│  Client ──► POST /auth/refresh ──► Auth Service             │
│             {refresh_token}                                  │
│                                                              │
│  Auth Service:                                               │
│  • Validate refresh token                                   │
│  • Check device binding                                     │
│  • Rotate refresh token (old one invalidated)               │
│  • Issue new access token                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 JWT Payload

```json
{
  "sub": "user_123",
  "vendor_code": "ABC",
  "roles": ["operator"],
  "permissions": ["read:batch", "write:barcode"],
  "assigned_batches": [1, 2, 3],
  "device_id": "device_xyz",
  "iat": 1735113600,
  "exp": 1735114500
}
```

---

## 4. Authorization Design

### 4.1 Row-Level Security (PostgreSQL)

```sql
-- Enable RLS on all barcode tables
ALTER TABLE bc_barcode ENABLE ROW LEVEL SECURITY;
ALTER TABLE bc_parent ENABLE ROW LEVEL SECURITY;
ALTER TABLE bc_batch ENABLE ROW LEVEL SECURITY;

-- Vendor isolation policy
CREATE POLICY vendor_isolation ON bc_barcode
  FOR ALL
  USING (vendor_code = current_setting('app.vendor_code')::text);

-- Batch assignment policy
CREATE POLICY batch_access ON bc_barcode
  FOR SELECT
  USING (
    batch_id IN (
      SELECT batch_id FROM user_batch_assignments 
      WHERE user_id = current_setting('app.user_id')::int
    )
  );

-- Write policy (only operators can write)
CREATE POLICY operator_write ON bc_barcode
  FOR INSERT
  WITH CHECK (
    current_setting('app.role')::text = 'operator'
    AND vendor_code = current_setting('app.vendor_code')::text
  );
```

### 4.2 Electric Shape Authorization

```yaml
# Electric configuration
shapes:
  batch_data:
    tables: [bc_parent, bc_barcode, bc_token]
    authorization:
      # JWT claim requirements
      require:
        - claim: vendor_code
          matches: :vendor_code
        - claim: assigned_batches
          contains: :batch_id
      # Write permissions
      write:
        require_role: operator
```

---

## 5. Data Encryption

### 5.1 Encryption at Rest

```
┌─────────────────────────────────────────────────────────────┐
│                 ENCRYPTION AT REST                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SERVER SIDE                                                 │
│  ───────────                                                 │
│  PostgreSQL:                                                 │
│  • Disk encryption (LUKS/AWS EBS encryption)                │
│  • Column-level encryption for sensitive fields             │
│  • Key management via AWS KMS / HashiCorp Vault             │
│                                                              │
│  CLIENT SIDE                                                 │
│  ───────────                                                 │
│  SQLite (Mobile/Desktop):                                   │
│  • SQLCipher encryption                                     │
│  • Key derived from user PIN + device key                   │
│  • Key stored in Keychain (iOS) / Keystore (Android)        │
│                                                              │
│  IndexedDB (Web):                                           │
│  • Web Crypto API encryption                                │
│  • Key derived from password + PBKDF2                       │
│  • Consider not storing sensitive data in web version       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Encryption in Transit

```
All connections MUST use TLS 1.3

HTTPS endpoints:
• Certificate: Let's Encrypt / AWS ACM
• HSTS: max-age=31536000; includeSubDomains
• Content-Security-Policy: strict

WebSocket:
• wss:// only (no ws://)
• Same certificate as HTTPS

Mobile certificate pinning:
• Pin to leaf certificate
• Include backup pin for rotation
```

### 5.3 Field-Level Encryption (Optional)

```python
# For extra-sensitive fields (e.g., token values)
from cryptography.fernet import Fernet

class EncryptedField:
    def __init__(self, key: bytes):
        self.fernet = Fernet(key)
    
    def encrypt(self, value: str) -> str:
        return self.fernet.encrypt(value.encode()).decode()
    
    def decrypt(self, encrypted: str) -> str:
        return self.fernet.decrypt(encrypted.encode()).decode()

# Usage in model
bc_token.token = encrypted_field.encrypt(raw_token)
```

---

## 6. Security Controls Matrix

| Control | Server | Mobile | Web | Priority |
|---------|--------|--------|-----|----------|
| TLS 1.3 | ✅ | ✅ | ✅ | CRITICAL |
| JWT auth | ✅ | ✅ | ✅ | CRITICAL |
| RLS | ✅ | N/A | N/A | CRITICAL |
| SQLite encryption | N/A | ✅ | N/A | HIGH |
| Certificate pinning | N/A | ✅ | N/A | HIGH |
| Rate limiting | ✅ | N/A | N/A | HIGH |
| Audit logging | ✅ | ✅ | ✅ | MEDIUM |
| WAF | ✅ | N/A | N/A | MEDIUM |
| Field encryption | Optional | Optional | Optional | LOW |

---

## 7. Incident Response

### 7.1 Security Incident Types

| Type | Severity | Response Time |
|------|----------|---------------|
| Data breach | CRITICAL | < 1 hour |
| Token compromise | HIGH | < 4 hours |
| DDoS attack | HIGH | < 1 hour |
| Unauthorized access | MEDIUM | < 24 hours |

### 7.2 Remote Wipe Capability

```
IF device_compromised:
  1. Revoke all tokens for device_id
  2. Add device_id to blocklist
  3. Trigger remote wipe via push notification
  4. Client deletes local SQLite DB
  5. Log incident to security team
```

---

## 8. Compliance Considerations

| Standard | Requirement | Implementation |
|----------|-------------|----------------|
| GDPR | Data minimization | Sync only needed data |
| GDPR | Right to erasure | Cascade delete in sync |
| ISO 27001 | Access control | RLS + RBAC |
| SOC 2 | Audit trails | All ops logged |
| PCI DSS | Encryption | TLS + SQLCipher |

---

## 9. Security Checklist

### Pre-Production

- [ ] Penetration testing completed
- [ ] Security code review done
- [ ] Dependency vulnerability scan
- [ ] TLS configuration validated
- [ ] JWT implementation reviewed
- [ ] RLS policies tested
- [ ] Rate limiting configured
- [ ] WAF rules deployed

### Ongoing

- [ ] Weekly vulnerability scans
- [ ] Monthly access reviews
- [ ] Quarterly penetration tests
- [ ] Annual security audit

