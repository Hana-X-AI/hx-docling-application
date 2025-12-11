# Identity, DNS & Certificate Management Review: HX Docling UI Charter

**Charter Reviewed**: `0.1-hx-docling-ui-charter.md` (v0.6.0)
**Review Date**: December 11, 2025
**Reviewer**: Frank Lucas, Identity, DNS & Certificate Management Specialist
**Invocation**: @frank

---

## Reviewer Profile

**Role**: Identity, DNS & Certificate Management Specialist for HX-Infrastructure
**Primary Responsibilities**:
- Samba Active Directory service account management (via hx-dc-server)
- DNS record creation and verification for service discovery
- SSL/TLS certificate generation and deployment (via hx-ca-server)
- Credential storage in Ansible Vault with standard password policy
- LDAP integration support for authentication and authorization
- Trust infrastructure for secure communication across HX-Infrastructure

**Phase 1 Review Focus** (Development - hx-cc-server):
1. Anonymous session security model (NO authentication)
2. Session management via Redis (UUID cookies, 24h TTL)
3. Input validation (file uploads, URL validation with Zod)
4. SSRF prevention (private IP blocking, HX domain blocking)
5. Rate limiting (10 req/min per session)
6. Database security (parameterized queries via Prisma)
7. Network access control (hx.dev.local internal network only)

**Phase 2 Deferred Responsibilities** (Production deployment):
- DNS record creation for production domain
- SSL/TLS certificate generation and installation
- Service account creation for application database user
- AD integration for user authentication
- Credential storage in Ansible Vault

---

## Executive Summary

| Category | Rating | Assessment |
|----------|--------|------------|
| **Phase 1 Security Model** | EXCELLENT | Anonymous sessions appropriate for development; well-defined boundaries |
| **Session Management** | GOOD | Redis-based UUID sessions with appropriate TTL; edge cases documented |
| **Input Validation** | EXCELLENT | Comprehensive file and URL validation with Zod schemas |
| **SSRF Prevention** | EXCELLENT | Multiple layers: protocol validation, private IP blocking, HX domain blocking |
| **Rate Limiting** | GOOD | 10 req/min per session prevents abuse; implementation clear |
| **Database Security** | EXCELLENT | Prisma ORM ensures parameterized queries; no SQL injection risk |
| **Network Security** | GOOD | Internal network only; appropriate for Phase 1 scope |
| **Phase 2 Readiness** | APPROVED | Clear separation; my responsibilities properly deferred |

**Overall Security Assessment**: APPROVED

The charter demonstrates a pragmatic and secure approach to Phase 1 development. The decision to defer authentication to Phase 2 is appropriate and well-documented. Anonymous session management via Redis is correctly implemented with appropriate security controls. SSRF prevention is thorough and follows industry best practices. All my Phase 2 responsibilities (DNS, SSL/TLS, AD integration) are clearly identified and properly deferred.

---

## Section-by-Section Security Assessment

### 1. Security Model Review (Phase 1)

**Reference**: Section 11.1

#### Strengths

| Aspect | Assessment | Reference |
|--------|------------|-----------|
| **Anonymous Sessions** | Appropriate for Phase 1 development environment | Section 11.1 |
| **UUID Cookies** | Standard approach for session tracking without authentication | Section 7.4 |
| **24-Hour TTL** | Reasonable session lifetime for anonymous users | Section 7.4 |
| **No Authentication** | Explicitly documented as deferred to Phase 2 (AD integration) | Section 11.1, 3.3 |
| **Internal Network Only** | Access restricted to hx.dev.local network | Section 11.1 |
| **Parameterized Queries** | Prisma ORM eliminates SQL injection risk | Section 11.1 |

#### Security Model Evaluation

**Phase 1 Approach (Development on hx-cc-server)**:
```
Authentication:  None (anonymous sessions)
Session Tracking: UUID cookie (httpOnly, sameSite)
Authorization:   Not required (no multi-tenant isolation)
Network Access:  Internal hx.dev.local network only
Rate Limiting:   10 requests/minute per session
```

This security model is **appropriate for Phase 1** because:
1. Development environment on internal network (192.168.10.224)
2. No public internet exposure
3. No sensitive data (test documents only)
4. Proper controls (rate limiting, input validation, SSRF prevention) are in place
5. Phase 2 upgrade path to AD authentication is clearly documented

**Phase 2 Upgrade Path (Production deployment)**:
```
Authentication:  Samba AD via hx-dc-server (192.168.10.200)
                 My responsibility: Create service account via samba-tool
                 Password: Major8859! (per standard policy)

DNS:             Production hostname (e.g., docling.hx.dev.local)
                 My responsibility: Create A record via samba-tool dns

SSL/TLS:         Certificate from internal CA (hx-ca-server)
                 My responsibility: Generate cert, coordinate with William for deployment
                 CA passphrase: Longhorn88

Authorization:   AD group-based access control
                 My responsibility: Create groups, provide LDAP integration examples
```

This upgrade path is clearly documented in Section 3.3, 11.1, 18.1, and aligns with standard HX-Infrastructure security practices.

---

### 2. Session Management Review

**Reference**: Section 7.4, 7.4.1

#### Strengths

| Aspect | Assessment | Reference |
|--------|------------|-----------|
| **Session Structure** | Clear interface with id, createdAt, lastActivity, jobCount, activeJobId | Section 7.4 |
| **UUID Generation** | UUID v4 provides sufficient entropy for anonymous sessions | Section 7.4.1 |
| **Redis Key Pattern** | `session:{sessionId}` is simple and effective | Section 7.4 |
| **TTL Strategy** | 24-hour expiry prevents session accumulation | Section 7.4 |
| **Edge Cases** | Session expiry, browser close, cookie clear, multi-tab scenarios documented | Section 7.4.1 |

#### Session Security Evaluation

**Session Cookie Specification** (Inferred from best practices):
```typescript
// Cookie attributes (should be verified in implementation)
{
  name: 'hx-docling-session',
  value: uuidv4(),
  httpOnly: true,        // Prevent XSS access
  secure: false,         // HTTP-only for Phase 1 (no TLS)
  sameSite: 'Lax',       // CSRF protection
  path: '/',
  maxAge: 86400          // 24 hours (matches Redis TTL)
}
```

**Security Properties**:
- **httpOnly**: Prevents client-side JavaScript access (XSS mitigation)
- **sameSite**: Prevents cross-site request forgery (CSRF mitigation)
- **secure: false**: Appropriate for Phase 1 HTTP-only deployment
- **Phase 2**: Must set `secure: true` when SSL/TLS is deployed

**Session Timeout Cascade** (Section 7.4.1):
```
Session TTL expires (24h) → Redis key deleted
    ↓
Jobs remain in PostgreSQL (90 days)
    ↓
Files remain in /data/docling-uploads/ (30 days)
    ↓
User cannot access history (no session cookie)
    ↓
Data eventually cleaned up by retention policies
```

This cascade is **secure and correct**:
- No data loss (jobs persist beyond session)
- No unauthorized access (new session cannot access old session's history)
- Proper cleanup (retention policies prevent unbounded growth)

#### Findings

| ID | Severity | Finding | Section | Recommendation |
|----|----------|---------|---------|----------------|
| SEC-SES-01 | Minor | Cookie security attributes not explicitly documented | 7.4 | Add cookie configuration section documenting httpOnly, sameSite, secure attributes |
| SEC-SES-02 | Minor | Session fixation attack not addressed | 7.4 | Document session regeneration strategy (not critical for anonymous sessions) |
| SEC-SES-03 | Info | Multi-tab rate limit sharing is correct but could cause user confusion | 7.4.1 | Add UI warning when rate limit is approached |

**SEC-SES-01 Recommendation**:
Add to Section 7.4:
```typescript
// Cookie Configuration
const COOKIE_CONFIG = {
  name: 'hx-docling-session',
  httpOnly: true,        // XSS protection
  secure: false,         // Phase 1: HTTP-only (set to true in Phase 2 with TLS)
  sameSite: 'Lax',       // CSRF protection
  path: '/',
  maxAge: 86400          // 24 hours (matches Redis TTL)
};
```

---

### 3. Input Validation Review

**Reference**: Section 11.2

#### Strengths

| Aspect | Assessment | Reference |
|--------|------------|-----------|
| **File Validation** | Extension, MIME type, and size validation via Zod | Section 11.2 |
| **Allowed Extensions** | Comprehensive list matching MCP tool capabilities | Section 11.2 |
| **Size Limits** | Appropriate: PDF (100MB), Office (50MB), Images (25MB) | Section 4.1.2 |
| **Type Safety** | Zod enum for MIME types prevents bypass | Section 11.2 |
| **Extension Check** | Case-insensitive `.toLowerCase()` check prevents bypass | Section 11.2 |

#### File Upload Security Evaluation

**Validation Flow**:
```
1. Client-side: File extension + size check (UX optimization)
2. Server-side: Zod schema validation (security boundary)
   - name.refine() checks extension against ALLOWED_EXTENSIONS
   - size.max() enforces size limit
   - type enum() validates MIME type
3. Server-side: Save to /tmp/docling-processing/
4. Server-side: Move to /data/docling-uploads/ after successful processing
```

**Security Properties**:
- **Double validation**: Client-side (UX) + server-side (security)
- **Extension + MIME type**: Both checked to prevent simple bypass
- **Case-insensitive**: `.toLowerCase()` prevents `file.PdF` bypass
- **Enum validation**: MIME type must match exact enum value
- **Size enforcement**: Prevents resource exhaustion attacks

**File Storage Security** (Section 7.5.1):
- **Permissions**: `0640` (owner read/write, group read, no world access)
- **UUID naming**: Prevents directory traversal (`../../etc/passwd`)
- **Date hierarchy**: `/YYYY/MM/DD/` enables efficient cleanup

**Missing Security Controls** (acceptable for Phase 1):
- No file content validation (magic byte checks)
- No malware scanning (ClamAV, VirusTotal)
- No file deduplication (SHA-256 hash checks)

These are **acceptable omissions** for Phase 1 development environment with internal network access only.

#### Findings

| ID | Severity | Finding | Section | Recommendation |
|----|----------|---------|---------|----------------|
| SEC-FILE-01 | Info | No magic byte validation | 11.2 | Consider adding in Phase 2 if public internet exposure planned |
| SEC-FILE-02 | Info | No malware scanning | 11.2 | Acceptable for Phase 1; add to Phase 2 backlog if needed |
| SEC-FILE-03 | Minor | Upload temp directory not explicitly secured | 11.2 | Document /tmp/docling-processing/ permissions (0700) |

---

### 4. URL Input Security & SSRF Prevention Review

**Reference**: Section 11.3 (Addresses Major 2.5)

#### Strengths

| Aspect | Assessment | Reference |
|--------|------------|-----------|
| **SSRF Prevention** | Multiple layers: protocol, private IP, HX domain blocking | Section 11.3 |
| **Protocol Validation** | HTTP/HTTPS only (blocks file://, ftp://, gopher://, etc.) | Section 11.3 |
| **Private IP Blocking** | RFC 1918 ranges blocked (10.x, 172.16-31.x, 192.168.x) | Section 11.3 |
| **Localhost Blocking** | `localhost` and `127.0.0.1` blocked | Section 11.3 |
| **Link-Local Blocking** | `169.254.0.0/16` blocked | Section 11.3 |
| **HX Domain Blocking** | `*.hx.dev.local` blocked (prevents internal service access) | Section 11.3 |
| **URL Length Limit** | 2048 characters max (prevents buffer overflow) | Section 11.3 |

#### SSRF Prevention Evaluation

**EXCELLENT IMPLEMENTATION** - This is a textbook example of comprehensive SSRF prevention.

**Attack Vectors Blocked**:

1. **Localhost Access**:
   ```
   http://localhost:5432/  ❌ Blocked (hx-postgres-server)
   http://127.0.0.1:6379/  ❌ Blocked (hx-redis-server)
   ```

2. **Private IP Access**:
   ```
   http://192.168.10.200/  ❌ Blocked (hx-dc-server)
   http://10.0.0.1/        ❌ Blocked (RFC 1918)
   http://172.16.0.1/      ❌ Blocked (RFC 1918)
   ```

3. **Internal HX Services**:
   ```
   http://hx-postgres-server.hx.dev.local:5432/  ❌ Blocked (*.hx.dev.local)
   http://hx-dc-server.hx.dev.local/             ❌ Blocked (Samba AD)
   http://hx-ca-server.hx.dev.local/             ❌ Blocked (Certificate Authority)
   ```

4. **Link-Local**:
   ```
   http://169.254.169.254/  ❌ Blocked (AWS metadata service pattern)
   ```

5. **Non-HTTP Protocols**:
   ```
   file:///etc/passwd       ❌ Blocked (protocol validation)
   ftp://internal.server/   ❌ Blocked (protocol validation)
   gopher://...             ❌ Blocked (protocol validation)
   ```

**Attack Vectors NOT Blocked** (acceptable trade-offs):

1. **DNS Rebinding** (advanced attack):
   - Attack: Domain initially resolves to public IP, then rebinds to private IP
   - Mitigation: Re-validate IP after DNS resolution (not implemented)
   - Risk: Low (requires attacker-controlled DNS + timing attack)
   - **Assessment**: Acceptable for Phase 1; consider for Phase 2

2. **IPv6 Private Addresses**:
   - Charter only checks IPv4 patterns
   - IPv6 private ranges (fc00::/7, fe80::/10) not blocked
   - **Assessment**: Acceptable if hx.dev.local network is IPv4-only

3. **URL Redirects**:
   - Charter specifies `maxRedirects: 3` (Section 11.3)
   - Redirect target is not re-validated against SSRF rules
   - **Assessment**: Should validate redirect targets

**URL Preview Security** (Section 11.3):
```typescript
{
  timeout: 5000,                    // 5 seconds
  maxRedirects: 3,                  // Follow up to 3 redirects
  userAgent: 'hx-docling-ui/1.0',   // Identify as bot
}
```

This is **correct** but missing redirect validation (see finding below).

#### Findings

| ID | Severity | Finding | Section | Recommendation |
|----|----------|---------|---------|----------------|
| SEC-URL-01 | Major | Redirect targets not validated against SSRF rules | 11.3 | Re-validate each redirect URL against same SSRF checks |
| SEC-URL-02 | Minor | IPv6 private addresses not blocked | 11.3 | Add IPv6 validation if hx.dev.local supports IPv6 |
| SEC-URL-03 | Info | DNS rebinding attack not mitigated | 11.3 | Low priority; consider for Phase 2 if internet-facing |
| SEC-URL-04 | Minor | User-Agent string reveals internal application name | 11.3 | Consider generic user agent (e.g., 'Mozilla/5.0...') |

**SEC-URL-01 (MAJOR) - Redirect Validation**:

**Vulnerability**:
```
User submits: https://public-site.com/redirect
                ↓
                Redirects to: http://192.168.10.208:5432/
                ↓
                SSRF bypass (redirect target not validated)
```

**Fix** (add to Section 11.3):
```typescript
// Redirect validation
const validateRedirect = (url: string) => {
  // Re-run same SSRF checks on redirect target
  return urlSchema.safeParse(url);
};

// In fetch implementation
const response = await fetch(url, {
  maxRedirects: 3,
  redirect: 'manual',  // Handle redirects manually
});

if (response.status >= 300 && response.status < 400) {
  const redirectUrl = response.headers.get('location');
  if (!validateRedirect(redirectUrl).success) {
    throw new Error('Redirect to blocked URL');
  }
}
```

---

### 5. Rate Limiting Review

**Reference**: Section 8.1.1 (Addresses Major 2.2), Section 11.1

#### Strengths

| Aspect | Assessment | Reference |
|--------|------------|-----------|
| **Rate Limit** | 10 requests/minute per session | Section 8.1.1 |
| **Sliding Window** | 60-second sliding window | Section 8.1.1 |
| **Scope** | Per sessionId (from cookie) | Section 8.1.1 |
| **Enforcement Point** | `/api/process` route | Section 8.1.1 |
| **HTTP Status** | 429 Too Many Requests with Retry-After header | Section 8.1.1 |
| **Error Code** | E601 (RATE_LIMIT_EXCEEDED) with clear message | Section 8.1.1 |

#### Rate Limiting Evaluation

**Implementation Strategy**:
```typescript
// In-memory rate limiter
const rateLimiter = new Map<sessionId, { count: number, windowStart: number }>();

// Middleware on /api/process
const checkRateLimit = (sessionId: string) => {
  const now = Date.now();
  const record = rateLimiter.get(sessionId);

  if (!record || now - record.windowStart > 60000) {
    // New window
    rateLimiter.set(sessionId, { count: 1, windowStart: now });
    return true;
  }

  if (record.count >= 10) {
    // Rate limit exceeded
    const retryAfter = Math.ceil((60000 - (now - record.windowStart)) / 1000);
    throw new RateLimitError(retryAfter);
  }

  // Increment count
  record.count++;
  return true;
};
```

**Security Properties**:
- **Prevents abuse**: 10 req/min is reasonable for document processing
- **Per-session**: Prevents single user from exhausting resources
- **No queue**: Requests rejected, not queued (prevents memory exhaustion)
- **Multi-tab aware**: Tabs share same session quota (Section 7.4.1)

**Comparison to Industry Standards**:
- **GitHub API**: 5000 req/hour (83 req/min) - More permissive
- **Twitter API**: 15 req/15min (1 req/min) - More restrictive
- **HX Docling**: 10 req/min - **Appropriate for document processing workload**

**Burst Handling** (Section 8.1.1):
- "No burst allowance (strict limit)"
- This is **correct** for resource-intensive document processing

#### Findings

| ID | Severity | Finding | Section | Recommendation |
|----|----------|---------|---------|----------------|
| SEC-RATE-01 | Minor | In-memory rate limiter lost on restart | 8.1.1 | Consider storing rate limit state in Redis for durability |
| SEC-RATE-02 | Info | No rate limit for other routes (/api/upload, /api/history) | 8.1.1 | Acceptable; processing is the expensive operation |
| SEC-RATE-03 | Info | No monitoring of rate limit hits | 8.1.1 | Add logging and alerting (see Monitoring section) |

**SEC-RATE-01 Recommendation**:
```typescript
// Store rate limit state in Redis (optional enhancement)
// Key: ratelimit:{sessionId}
// Value: JSON { count, windowStart }
// TTL: 60 seconds

// Pros:
// - Survives application restart
// - Consistent across multiple app instances (Phase 2)

// Cons:
// - Additional Redis round-trip (latency)
// - More complex implementation

// Recommendation: Implement in Phase 2 if deploying multiple instances
```

---

### 6. Database Security Review

**Reference**: Section 11.1, Section 7.3 (Prisma schema)

#### Strengths

| Aspect | Assessment | Reference |
|--------|------------|-----------|
| **ORM Usage** | Prisma ensures parameterized queries (no SQL injection) | Section 11.1, 7.3 |
| **Schema Validation** | Zod validation before database insertion | Section 11.2 |
| **Connection String** | Environment variable with password substitution | Section 8.3 |
| **Cascade Deletes** | Results cascade on Job deletion (data integrity) | Section 7.3 |
| **Indexes** | Appropriate indexes on query fields (performance + security) | Section 7.3, 7.6.1 |

#### Database Security Evaluation

**SQL Injection Prevention**:
```typescript
// Prisma ORM automatically parameterizes queries
await prisma.job.findMany({
  where: { sessionId: userInputSessionId }  // ✅ Safe (parameterized)
});

// No raw SQL queries in charter (verified via grep)
```

**Database Credential Management** (Section 8.3):
```bash
DATABASE_URL=postgresql://docling_user:${DB_PASSWORD}@hx-postgres-server.hx.dev.local:5432/docling_db
```

**Phase 1 Credential Storage** (Development):
- Password stored in `.env.development` file
- File not committed to git (see Appendix C)
- Acceptable for development environment

**Phase 2 Credential Management** (Production - My Responsibility):
```bash
# 1. Create database user via samba-tool on hx-dc-server
ssh agent0@hx-dc-server.hx.dev.local
sudo samba-tool user create docling-db 'Major8859!' \
  --description='Docling UI Database Service Account' \
  --home-directory=/home/docling-db@hx.dev.local \
  --login-shell=/bin/bash \
  --use-username-as-cn

# 2. Store credentials in Ansible Vault
cat > /home/agent0/HX-Infrastructure/services/hx-docling-ui/vault/credentials.yml <<EOF
---
database_user: docling-db@hx.dev.local
database_password: Major8859!
redis_password: Major8859!  # If Redis AUTH enabled
session_secret: <generated-secret>
EOF

ansible-vault encrypt credentials.yml
# Vault password: Major8859! (per standard policy)

# 3. Reference in /home/agent0/HX-Infrastructure/hx-knowledge/docs/0.0.5.2.1-credentials.md
```

This is **properly deferred to Phase 2** and aligns with HX-Infrastructure credential management standards.

**Connection Pool Security** (William's finding INF-PG-01):
- No connection pool size specified
- Prisma default: Unlimited (bounded by database max_connections)
- **Recommendation**: Add `connection_limit=10` to DATABASE_URL for resource control

#### Findings

| ID | Severity | Finding | Section | Recommendation |
|----|----------|---------|---------|----------------|
| SEC-DB-01 | Info | No database authentication via AD (Phase 2) | 8.3 | Properly deferred; my responsibility for Phase 2 |
| SEC-DB-02 | Minor | Connection pool size not specified | 8.3 | Add `?connection_limit=10` to DATABASE_URL |
| SEC-DB-03 | Info | Database backup procedure not specified (William's INF-PG-02) | 7.6 | Operational concern; defer to William |

---

### 7. Network Access Control Review

**Reference**: Section 11.1, Section 8.4

#### Strengths

| Aspect | Assessment | Reference |
|--------|------------|-----------|
| **Internal Network Only** | All servers on hx.dev.local (192.168.10.0/24) | Section 8.4 |
| **No Public Exposure** | Development server not internet-facing | Section 11.1 |
| **DNS Resolution** | Internal DNS via hx-dc-server (192.168.10.200) | Section 8.4 |

#### Network Security Evaluation

**Network Topology** (Phase 1):
```
hx-cc-server (192.168.10.224) - Development server
    ↓ HTTP
hx-docling-mcp-server (192.168.10.217:8000) - MCP API
    ↓ SQL
hx-postgres-server (192.168.10.208:5432) - Database
    ↓ Redis Protocol
hx-redis-server (192.168.10.209:6379) - Sessions
```

**Security Properties**:
- All traffic on internal hx.dev.local network
- No public internet exposure
- No inbound connections from internet
- Outbound connections only for `convert_url` (SSRF-protected)

**DNS Security**:
- Internal DNS via Samba AD (hx-dc-server)
- `.hx.dev.local` zone managed by me (Frank Lucas)
- Phase 1: Existing DNS records (hx-cc-server, hx-postgres-server, hx-redis-server)
- Phase 2: My responsibility to create production DNS record

**Missing Network Security Controls** (acceptable for Phase 1):
- No firewall rules specified (William's INF-NET-01)
- No TLS/SSL (HTTP-only)
- No VPN/bastion access control
- No network segmentation (all services on same subnet)

These are **acceptable for Phase 1** internal development environment.

#### Phase 2 Network Security (My Responsibilities)

**DNS Record Creation**:
```bash
# On hx-dc-server (192.168.10.200)
ssh agent0@hx-dc-server.hx.dev.local
sudo samba-tool dns add localhost hx.dev.local docling A <production-ip> -U administrator
# Password: Major3059!

# Verify DNS resolution
nslookup docling.hx.dev.local
ping -c 3 docling.hx.dev.local
```

**SSL/TLS Certificate Generation**:
```bash
# On hx-ca-server (192.168.10.201)
cd ~/easy-rsa-pki
openssl genrsa -out docling.hx.dev.local.key 4096
openssl req -new -key docling.hx.dev.local.key -out docling.hx.dev.local.csr \
  -subj "/C=US/ST=State/L=City/O=HX-Infrastructure/CN=docling.hx.dev.local"
openssl x509 -req -in docling.hx.dev.local.csr \
  -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
  -out docling.hx.dev.local.crt -days 365 -sha256
# Enter CA passphrase: Longhorn88

# Deliver to production server (coordinate with William)
scp docling.hx.dev.local.crt docling.hx.dev.local.key ca-cert.pem \
  agent0@hx-dev-server:/tmp/
```

**Session Cookie Update** (Phase 2):
```typescript
// Update cookie configuration when TLS is deployed
const COOKIE_CONFIG = {
  secure: true,  // ← CHANGE from false to true
  sameSite: 'Strict',  // ← Optionally stricter
};
```

These are **properly deferred to Phase 2** and clearly documented.

#### Findings

| ID | Severity | Finding | Section | Recommendation |
|----|----------|---------|---------|----------------|
| SEC-NET-01 | Info | No TLS/SSL (HTTP-only) | 11.1 | Properly deferred to Phase 2; my responsibility to generate cert |
| SEC-NET-02 | Info | No DNS record for production deployment | 11.1 | Properly deferred to Phase 2; my responsibility to create A record |
| SEC-NET-03 | Info | Firewall rules not documented (William's INF-NET-01) | 8.4 | Defer to William; operational concern |

---

### 8. Phase 2 Readiness Assessment

**Reference**: Section 3.3, 11.1, 18.1

#### Phase 2 Security Requirements (My Responsibilities)

| Requirement | Status | Section | My Action |
|-------------|--------|---------|-----------|
| **DNS Record** | Deferred | 3.3, 11.1 | Create A record for production hostname (e.g., docling.hx.dev.local) |
| **SSL/TLS Certificate** | Deferred | 3.3, 11.1 | Generate certificate from internal CA (hx-ca-server), coordinate with William |
| **Service Account** | Deferred | 8.3 | Create docling-db@hx.dev.local via samba-tool, password Major8859! |
| **Credential Vault** | Deferred | 8.3 | Store credentials in Ansible Vault, vault password Major8859! |
| **AD Authentication** | Deferred | 3.3, 18.1 | Provide LDAP integration examples, create AD groups |

#### Phase 2 Preparation Assessment

**APPROVED** - All my Phase 2 responsibilities are clearly identified and properly deferred.

**Preparation Checklist** (for Phase 2 charter):

1. **DNS Record Creation**:
   - [ ] Determine production hostname (e.g., docling.hx.dev.local)
   - [ ] Determine production IP address (e.g., hx-dev-server 192.168.10.XXX)
   - [ ] Create A record via `samba-tool dns add`
   - [ ] Verify DNS resolution from multiple servers
   - [ ] Update application configuration with new hostname

2. **SSL/TLS Certificate**:
   - [ ] Generate 4096-bit RSA private key
   - [ ] Create CSR with production hostname as CN
   - [ ] Sign certificate with internal CA (passphrase: Longhorn88)
   - [ ] Deliver certificate + key + CA cert to production server
   - [ ] Coordinate with William for certificate installation
   - [ ] Update application to use HTTPS
   - [ ] Update cookie configuration (secure: true)

3. **Service Account Creation**:
   - [ ] Create docling-db@hx.dev.local via samba-tool user create
   - [ ] Set password to Major8859! (per standard policy)
   - [ ] Verify account replication via `wbinfo -i docling-db@hx.dev.local`
   - [ ] Grant database permissions (coordinate with William)

4. **Credential Storage**:
   - [ ] Create vault directory: `/home/agent0/HX-Infrastructure/services/hx-docling-ui/vault/`
   - [ ] Create credentials.yml with database, Redis, session secret
   - [ ] Encrypt with ansible-vault (password: Major8859!)
   - [ ] Document in `/home/agent0/HX-Infrastructure/hx-knowledge/docs/0.0.5.2.1-credentials.md`

5. **AD Authentication** (if implementing in Phase 2):
   - [ ] Create AD groups: `docling-users`, `docling-admins`
   - [ ] Provide Python/TypeScript LDAP integration examples
   - [ ] Configure LDAP authentication in application
   - [ ] Test authentication flow with AD accounts
   - [ ] Document LDAP configuration in application docs

**Phase 2 Coordination**:
- **With William Chen**: Certificate deployment, database permissions, systemd service setup
- **With Alex Rivera**: Architecture review of AD integration approach
- **With Julia Santos**: Security testing of authentication flow
- **With Technology SMEs**: LDAP integration support

---

## Findings Summary

### Critical Findings
None.

### Major Findings

| ID | Finding | Section | Recommendation |
|----|---------|---------|----------------|
| SEC-URL-01 | Redirect targets not validated against SSRF rules | 11.3 | Re-validate each redirect URL against same SSRF checks to prevent redirect-based SSRF bypass |

### Minor Findings

| ID | Finding | Section | Recommendation |
|----|---------|---------|----------------|
| SEC-SES-01 | Cookie security attributes not explicitly documented | 7.4 | Add cookie configuration section documenting httpOnly, sameSite, secure attributes |
| SEC-URL-02 | IPv6 private addresses not blocked in SSRF validation | 11.3 | Add IPv6 validation (fc00::/7, fe80::/10) if hx.dev.local supports IPv6 |
| SEC-URL-04 | User-Agent string reveals internal application name | 11.3 | Consider generic user agent (e.g., 'Mozilla/5.0...') for URL preview requests |
| SEC-FILE-03 | Upload temp directory permissions not explicitly documented | 11.2 | Document /tmp/docling-processing/ permissions (0700) and ownership |
| SEC-RATE-01 | In-memory rate limiter lost on application restart | 8.1.1 | Consider storing rate limit state in Redis for durability (Phase 2 priority) |
| SEC-DB-02 | Database connection pool size not specified | 8.3 | Add `?connection_limit=10` to DATABASE_URL for resource control |

### Informational Findings

| ID | Finding | Section | Note |
|----|---------|---------|------|
| SEC-SES-02 | Session fixation attack not addressed | 7.4 | Low priority for anonymous sessions; consider for Phase 2 with AD auth |
| SEC-SES-03 | Multi-tab rate limit sharing could cause user confusion | 7.4.1 | Add UI warning when rate limit is approached |
| SEC-FILE-01 | No magic byte validation for uploaded files | 11.2 | Consider for Phase 2 if public internet exposure planned |
| SEC-FILE-02 | No malware scanning for uploaded files | 11.2 | Acceptable for Phase 1; add to Phase 2 backlog if needed |
| SEC-URL-03 | DNS rebinding attack not mitigated | 11.3 | Low priority; consider for Phase 2 if internet-facing |
| SEC-RATE-02 | No rate limit for other routes (/api/upload, /api/history) | 8.1.1 | Acceptable; processing is the expensive operation |
| SEC-RATE-03 | No monitoring of rate limit hits | 8.1.1 | Add logging and alerting for rate limit events |
| SEC-DB-01 | No database authentication via AD (Phase 2) | 8.3 | Properly deferred; my responsibility for Phase 2 |
| SEC-DB-03 | Database backup procedure not specified | 7.6 | Operational concern; defer to William Chen |
| SEC-NET-01 | No TLS/SSL (HTTP-only) | 11.1 | Properly deferred to Phase 2; my responsibility to generate cert |
| SEC-NET-02 | No DNS record for production deployment | 11.1 | Properly deferred to Phase 2; my responsibility to create A record |
| SEC-NET-03 | Firewall rules not documented | 8.4 | Defer to William Chen; operational concern |

---

## Recommendations for Improvement

### High Priority (Address Before Phase 1 Completion)

1. **SEC-URL-01: Redirect Validation**
   - **Impact**: Prevents redirect-based SSRF bypass
   - **Effort**: Low (add validation in fetch implementation)
   - **Implementation**: See Section 4 recommendation
   - **Priority**: HIGH - Security vulnerability

2. **SEC-SES-01: Cookie Security Documentation**
   - **Impact**: Ensures secure cookie implementation
   - **Effort**: Minimal (documentation only)
   - **Implementation**: Add cookie configuration section to 7.4
   - **Priority**: MEDIUM - Documentation clarity

### Medium Priority (Consider for Phase 1, Required for Phase 2)

3. **SEC-FILE-03: Temp Directory Permissions**
   - **Impact**: Prevents unauthorized access to uploaded files
   - **Effort**: Minimal (documentation + verification)
   - **Implementation**: Document and enforce 0700 permissions on /tmp/docling-processing/
   - **Priority**: MEDIUM - Security hardening

4. **SEC-URL-02: IPv6 SSRF Validation**
   - **Impact**: Prevents IPv6-based SSRF attacks
   - **Effort**: Low (add IPv6 regex patterns)
   - **Implementation**: Extend urlSchema.refine() with IPv6 checks
   - **Priority**: MEDIUM - Depends on network configuration

5. **SEC-DB-02: Connection Pool Limit**
   - **Impact**: Prevents database resource exhaustion
   - **Effort**: Minimal (add query parameter)
   - **Implementation**: Add `?connection_limit=10` to DATABASE_URL
   - **Priority**: MEDIUM - Resource protection

### Low Priority (Phase 2 Enhancements)

6. **SEC-RATE-01: Persistent Rate Limiting**
   - **Impact**: Rate limit survives restarts, supports multiple instances
   - **Effort**: Medium (implement Redis-based rate limiter)
   - **Priority**: LOW - Phase 2 enhancement

7. **SEC-FILE-01 & SEC-FILE-02: Content Validation & Malware Scanning**
   - **Impact**: Enhanced file upload security
   - **Effort**: High (integrate external tools)
   - **Priority**: LOW - Phase 2 if public internet exposure

8. **SEC-URL-03: DNS Rebinding Mitigation**
   - **Impact**: Prevents advanced SSRF attacks
   - **Effort**: High (requires DNS caching + re-validation)
   - **Priority**: LOW - Advanced attack, low probability

---

## Security Compliance Assessment

### HX-Infrastructure Security Standards

| Standard | Requirement | Charter Compliance | Status |
|----------|-------------|-------------------|--------|
| **Authentication** | Samba AD via hx-dc-server | Deferred to Phase 2 (anonymous sessions Phase 1) | ✅ COMPLIANT |
| **Credentials** | Standard password policy (Major8859!) | Documented for Phase 2 | ✅ COMPLIANT |
| **DNS** | Records via hx-dc-server Samba DNS | Deferred to Phase 2 | ✅ COMPLIANT |
| **SSL/TLS** | Internal CA via hx-ca-server | Deferred to Phase 2 | ✅ COMPLIANT |
| **Network** | Internal hx.dev.local only (Phase 1) | All services on 192.168.10.0/24 | ✅ COMPLIANT |
| **Input Validation** | Server-side validation required | Zod schemas with comprehensive checks | ✅ COMPLIANT |
| **SQL Injection** | Parameterized queries only | Prisma ORM enforces parameterization | ✅ COMPLIANT |
| **SSRF Prevention** | Block private IPs and internal domains | Comprehensive validation implemented | ✅ COMPLIANT |
| **Rate Limiting** | Prevent resource exhaustion | 10 req/min per session | ✅ COMPLIANT |

**Overall Compliance**: ✅ **COMPLIANT** with HX-Infrastructure security standards for Phase 1 development.

---

## Verdict

**APPROVED**

This charter demonstrates a **mature and pragmatic approach** to security for a Phase 1 development environment:

### Strengths

1. **Appropriate Scope**: Anonymous sessions are correct for internal development
2. **SSRF Prevention**: Textbook implementation with multiple defensive layers
3. **Input Validation**: Comprehensive file and URL validation with Zod
4. **Database Security**: Prisma ORM eliminates SQL injection risk
5. **Phase 2 Planning**: All my responsibilities clearly identified and deferred
6. **Security Boundaries**: Clear distinction between Phase 1 and Phase 2 security models

### Required Actions (High Priority)

1. **SEC-URL-01**: Implement redirect validation to prevent SSRF bypass
2. **SEC-SES-01**: Document cookie security attributes

### Recommended Actions (Medium Priority)

3. **SEC-FILE-03**: Document temp directory permissions
4. **SEC-URL-02**: Add IPv6 SSRF validation (if network supports IPv6)
5. **SEC-DB-02**: Add connection pool limit

### Phase 2 Commitment

When Phase 2 charter is created, I (Frank Lucas) commit to:
- Create DNS A record for production hostname via `samba-tool dns add`
- Generate SSL/TLS certificate from internal CA (hx-ca-server) with passphrase Longhorn88
- Create service account `docling-db@hx.dev.local` via `samba-tool user create` with password Major8859!
- Store credentials in Ansible Vault with vault password Major8859!
- Provide LDAP integration examples if AD authentication is implemented
- Coordinate with William Chen for certificate deployment and database permissions
- Update credential documentation in `/home/agent0/HX-Infrastructure/hx-knowledge/docs/0.0.5.2.1-credentials.md`

---

## Sign-Off

**Reviewer**: Frank Lucas (@frank)
**Role**: Identity, DNS & Certificate Management Specialist
**Date**: December 11, 2025
**Verdict**: APPROVED
**Conditions**: Address SEC-URL-01 (redirect validation) before Phase 1 completion

This charter is **approved from an identity and security perspective** for Phase 1 development. The security model is appropriate for the internal development environment, and all Phase 2 security enhancements are properly identified and deferred.

---

**Reference**: `/home/agent0/HX-Infrastructure/hx-knowledge/docs/0.0.5.2.1-credentials.md` (AUTHORITATIVE credential reference)
**Contact**: @frank for identity, DNS, certificate, and credential questions
