# 75. Design an Authentication and Authorization System (OAuth 2.0/SSO)

---

## 1. Functional Requirements (FR)

- **User registration/login**: Email+password, social login (Google, GitHub, Apple)
- **OAuth 2.0 provider**: Issue access tokens, refresh tokens; support authorization code, client credentials, PKCE flows
- **Single Sign-On (SSO)**: Login once, access multiple applications (SAML 2.0 and OIDC)
- **Multi-Factor Authentication (MFA)**: TOTP (Google Authenticator), SMS OTP, WebAuthn/passkeys
- **Role-Based Access Control (RBAC)**: Users have roles; roles have permissions
- **API key management**: Issue, rotate, revoke API keys for machine-to-machine auth
- **Session management**: Active session list, revoke sessions, device tracking
- **Password policies**: Minimum strength, breach detection (HaveIBeenPwned check), forced rotation

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Token validation < 5 ms (stateless JWT); login < 500 ms
- **High Availability**: 99.999% — auth down = entire platform down
- **Security**: Bcrypt/Argon2 password hashing; token encryption; rate limiting on login
- **Scale**: 1B+ users, 500K+ auth requests/sec
- **Compliance**: SOC 2, GDPR (right to delete), PCI DSS for payment-adjacent

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total users | 1B |
| Login requests / sec | 50K |
| Token validation / sec | 500K (every API call) |
| Token refresh / sec | 10K |
| Active sessions | 500M |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    TOKEN VALIDATION (HOT PATH)                         │
│                                                                        │
│  Every API Call:                                                       │
│  Client --> API Gateway --> JWT Validation (stateless, < 1ms)          │
│                |               |                                       │
│                |    +----------v-----------+                           │
│                |    | JWKS Cache           |                           │
│                |    | (public keys for     |                           │
│                |    |  signature verify)   |                           │
│                |    | Refreshed every 1h   |                           │
│                |    +----------+-----------+                           │
│                |               |                                       │
│                |    +----------v-----------+                           │
│                |    | Redis (optional      |                           │
│                |    |  revocation check:   |                           │
│                |    |  revoked_token:{jti})|                           │
│                |    +----------------------+                           │
│                |                                                       │
│                v (if valid)                                             │
│         Downstream Service (with user_id, roles from JWT)              │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                   AUTH SERVICE (LOGIN/SIGNUP PATH)                      │
│                                                                        │
│  Client --> API Gateway --> Auth Service                                │
│                                  |                                     │
│        +-------------------------+---------------------------+         │
│        |            |            |            |              |         │
│  +-----v-----+ +---v------+ +---v------+ +---v------+ +----v------+  │
│  | User      | | Token    | | MFA      | | Social   | | Session   |  │
│  | Service   | | Service  | | Service  | | Login    | | Manager   |  │
│  |           | |          | |          | | Service  | |           |  │
│  | - register| | - issue  | | - TOTP   | |          | | - track   |  │
│  | - login   | |   JWT    | |   verify | | - Google | |   active  |  │
│  | - password| | - refresh| | - SMS OTP| | - GitHub | |   sessions|  │
│  |   reset   | |   tokens | | - WebAuthn| | - Apple | | - device  |  │
│  | - profile | | - revoke | |   /passkey| | - SAML  | |   tracking|  │
│  |   mgmt    | |          | |          | |   IdPs   | | - revoke  |  │
│  +-----+-----+ +---+------+ +---+------+ +---+------+ +----+------+  │
│        |            |            |            |              |         │
│  +-----v---------------------------------------------------------+    │
│  |                    PostgreSQL                                  |    │
│  | users, refresh_tokens, user_roles, role_permissions,           |    │
│  | oauth_clients, social_accounts, sessions, login_audit_log      |    │
│  +----------------------------------------------------------------+   │
│                                                                        │
│  +-----v-----------+     +---------------+     +-------------------+   │
│  |    Redis         |     | Kafka          |     | Threat Detection |   │
│  | - sessions       |     | (auth-events)  |     | Service          |   │
│  | - revoked tokens |     |                |     | - credential     |   │
│  | - login attempts |     +-------+--------+     |   stuffing detect|   │
│  | - auth codes     |             |              | - impossible     |   │
│  | - permissions    |     +-------v--------+     |   travel (geo)   |   │
│  |   cache          |     | Audit Log      |     | - brute force    |   │
│  +------------------+     | Service        |     |   detection      |   │
│                           | (ClickHouse —  |     | - compromised    |   │
│                           |  login history,|     |   credential     |   │
│                           |  geo analysis, |     |   check (HIBP)   |   │
│                           |  anomalies)    |     +-------------------+  │
│                           +----------------+                           │
└────────────────────────────────────────────────────────────────────────┘
```
          |  tokens) |
          +---------+
```

### Component Deep Dive

#### OAuth 2.0 Authorization Code Flow with PKCE

```
For web/mobile apps authenticating users:

1. Client generates code_verifier (random 43-128 chars)
   code_challenge = BASE64URL(SHA256(code_verifier))

2. Client redirects user to auth server:
   GET /authorize?response_type=code&client_id=app-123
       &redirect_uri=https://app.com/callback
       &scope=openid profile email
       &code_challenge=E9Melhoa2OwvFrEMT...
       &code_challenge_method=S256
       &state=random-csrf-token

3. User authenticates (login form + MFA if enabled)

4. Auth server redirects back with authorization code:
   302 https://app.com/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=random-csrf-token

5. Client exchanges code for tokens:
   POST /token
   { grant_type: "authorization_code", code: "SplxlOBeZQQYbYS6WxSbIA",
     redirect_uri: "https://app.com/callback", client_id: "app-123",
     code_verifier: "dBjftJeZ4CVP-mB92K27uhbUJU1p..." }

6. Auth server validates code + verifier, returns tokens:
   { access_token: "eyJhbG...", token_type: "Bearer", expires_in: 3600,
     refresh_token: "tGzv3JOk...", id_token: "eyJhbG..." }

Why PKCE? Prevents authorization code interception attack.
Without PKCE: intercepted code can be exchanged for tokens.
With PKCE: code exchange requires code_verifier that only original client has.
```

#### JWT Token Structure and Validation

```
Access token (JWT):
  Header: { "alg": "RS256", "kid": "key-2026-03" }
  Payload: {
    "sub": "user-uuid",           // user ID
    "iss": "https://auth.example.com",
    "aud": "api.example.com",
    "exp": 1710403200,            // expiry (1 hour from issue)
    "iat": 1710399600,            // issued at
    "scope": "read write",
    "roles": ["admin", "editor"],
    "tenant_id": "tenant-uuid"
  }
  Signature: RSA-SHA256(header.payload, private_key)

Validation at API Gateway (stateless, < 1 ms):
  1. Decode JWT header -> get kid (key ID)
  2. Look up public key from cached JWKS (JSON Web Key Set)
  3. Verify signature with public key
  4. Check exp > current_time (not expired)
  5. Check iss matches expected issuer
  6. Check aud matches this service
  7. Extract roles/permissions for authorization
  
  No DB call needed! Stateless = scales infinitely for reads.

Token revocation challenge:
  JWT is stateless -> can't revoke a specific token (it's valid until exp).
  
  Solutions:
  a. Short expiry (15 min) + refresh tokens (long-lived, stored in DB)
     Revoke: delete refresh token from DB -> user must re-login after access token expires
  b. Token blacklist in Redis: SET revoked:{jti} "" EX {remaining_ttl}
     On validation: check if jti is in blacklist. Adds 1 Redis call per request.
     Acceptable at 500K/sec (Redis handles easily).
  c. Version counter: user has token_version in DB.
     JWT includes version. If JWT version < DB version -> rejected.
     On revocation: increment DB version.
```

#### RBAC — Role-Based Access Control

```
Hierarchy:
  User --> has --> Roles --> have --> Permissions

Example:
  User "Alice" -> roles: ["editor", "viewer"]
  Role "editor" -> permissions: ["article:create", "article:edit", "article:delete"]
  Role "viewer" -> permissions: ["article:read"]
  
Authorization check:
  Can Alice create an article?
  Alice's roles -> ["editor", "viewer"]
  "editor" permissions include "article:create" -> YES

Permission format: resource:action
  "article:create", "article:read", "user:admin", "billing:view"

Storage:
  user_roles: { user_id, role_name } (PostgreSQL, many-to-many)
  role_permissions: { role_name, permission } (PostgreSQL, many-to-many)
  
  Cache in JWT: roles array in token payload
  Cache in Redis: permissions:{user_id} -> SET of permission strings, TTL 300
  
  On role change: invalidate Redis cache + issue new JWT on next refresh
```

---

## 5. APIs

```http
POST /api/v1/auth/register
{ "email": "alice@example.com", "password": "SecureP@ss123", "name": "Alice" }
-> 201 { "user_id": "user-uuid", "email_verification_sent": true }

POST /api/v1/auth/login
{ "email": "alice@example.com", "password": "SecureP@ss123" }
-> 200 { "access_token": "eyJ...", "refresh_token": "tGz...", "expires_in": 3600 }
 OR 200 { "mfa_required": true, "mfa_token": "temp-token", "mfa_methods": ["totp","sms"] }

POST /api/v1/auth/mfa/verify
{ "mfa_token": "temp-token", "code": "123456", "method": "totp" }
-> 200 { "access_token": "eyJ...", "refresh_token": "tGz..." }

POST /api/v1/auth/token/refresh
{ "refresh_token": "tGz..." }
-> 200 { "access_token": "eyJ...(new)", "refresh_token": "tGz...(rotated)" }

POST /api/v1/auth/logout
Authorization: Bearer eyJ...
-> 200 { "logged_out": true } // revokes refresh token + blacklists access token

GET /api/v1/auth/sessions
-> { "sessions": [{ "device": "iPhone", "ip": "...", "last_active": "...", "current": true }] }
```

---

## 6. Data Model

### PostgreSQL

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY, email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255), -- bcrypt/argon2
    name VARCHAR(100), email_verified BOOLEAN DEFAULT FALSE,
    mfa_enabled BOOLEAN DEFAULT FALSE, mfa_secret VARCHAR(64),
    token_version INT DEFAULT 0, -- increment to invalidate all tokens
    status ENUM('active','suspended','deleted'), created_at TIMESTAMPTZ
);

CREATE TABLE refresh_tokens (
    token_id UUID PRIMARY KEY, user_id UUID NOT NULL,
    token_hash VARCHAR(64) NOT NULL, -- SHA256 of actual token
    device_info TEXT, ip_address INET,
    expires_at TIMESTAMPTZ NOT NULL, revoked BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_user (user_id), INDEX idx_token (token_hash)
);

CREATE TABLE user_roles (
    user_id UUID, role_name VARCHAR(50),
    PRIMARY KEY (user_id, role_name)
);

CREATE TABLE role_permissions (
    role_name VARCHAR(50), permission VARCHAR(100),
    PRIMARY KEY (role_name, permission)
);

CREATE TABLE oauth_clients (
    client_id VARCHAR(64) PRIMARY KEY, client_secret_hash VARCHAR(64),
    name VARCHAR(100), redirect_uris TEXT[], grant_types TEXT[],
    scopes TEXT[], active BOOLEAN DEFAULT TRUE
);
```

### Redis
```
session:{session_id}         -> Hash { user_id, device, ip, last_active }
revoked_token:{jti}          -> "", TTL = token remaining lifetime
permissions:{user_id}        -> SET of permission strings, TTL 300
login_attempts:{email}       -> INT (rate limit: max 5 per 15 min), TTL 900
auth_code:{code}             -> Hash { client_id, user_id, scope, code_challenge }, TTL 600
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Auth service down** | JWTs still validated at gateway (stateless); only login/refresh affected |
| **Redis down** | Token validation falls back to JWT-only (no revocation check); accept risk briefly |
| **Brute force** | Rate limit: 5 login attempts per 15 min per email; CAPTCHA after 3 failures |
| **Credential stuffing** | Check passwords against HaveIBeenPwned API; flag compromised accounts |
| **Token theft** | Short access token TTL (15 min); refresh token rotation; device binding |
| **Key compromise** | Key rotation: new signing key monthly; old keys valid for verification only |

---

## 8. Deep Dive

### Password Storage: Why Argon2id

```
bcrypt: good, but vulnerable to GPU attacks (fixed memory usage).
scrypt: better (memory-hard), but complex to tune.
Argon2id: winner of Password Hashing Competition.
  Memory-hard + CPU-hard. Configurable: time, memory, parallelism.
  Recommended: Argon2id with 64MB memory, 3 iterations, 4 threads.
  Result: each hash takes ~200ms and 64MB RAM -> GPU attacks infeasible.
```

### Refresh Token Rotation

```
On every token refresh:
  1. Validate current refresh_token
  2. Issue new access_token + NEW refresh_token
  3. Invalidate old refresh_token
  
If stolen old refresh_token is used:
  It's already invalidated -> request fails
  Detection: invalidated token reuse -> compromise detected -> revoke ALL tokens for user
  This is called "refresh token rotation with reuse detection".
```

### Credential Stuffing Defense

```
Attack: attacker uses leaked username/password lists from other breaches
  to try login on our platform. 1M attempts from botnet.

Defense layers:
  1. Rate limiting: max 5 failed attempts per email per 15 min (Redis counter)
  2. IP rate limiting: max 50 login attempts per IP per hour
  3. CAPTCHA after 3 failed attempts for same email
  4. Device fingerprinting: new device + correct password -> require email verification
  5. Breached password check: on registration AND login, check against HaveIBeenPwned API
     If password is in breach database: force password change before allowing login
  6. Anomaly detection: login from new country, new device, unusual hour
     Action: send push notification "Was this you?" with approve/deny

Implementation:
  Redis keys:
    login_fail:{email}           -> INT, TTL 900 (15 min)
    login_fail_ip:{ip}           -> INT, TTL 3600 (1 hour)  
    device_verified:{user}:{fp}  -> "1", TTL 30 days
  
  Flow:
    if login_fail:{email} > 5 -> return "Account temporarily locked"
    if login_fail_ip:{ip} > 50 -> return "Too many attempts from this IP"
    if login_fail:{email} > 2 -> require CAPTCHA
    if password correct AND device NOT verified -> send verification email
```

### Account Takeover (ATO) Detection

```
Signals that an account may be compromised:
  - Password changed + email changed within 5 minutes -> ATO in progress
  - Login from new country + immediate sensitive action (change password, add payment)
  - Session from unrecognized device + bulk data export
  
Action:
  1. Send notification to all other active sessions: "Security alert: new login detected"
  2. Require re-authentication for sensitive actions within 30 min of new device login
  3. If password AND email changed: lock account + send recovery to ORIGINAL email
  
Step-up authentication:
  Sensitive actions (change password, change email, view payment info, export data)
  require re-entering password even if session is valid.
  This prevents an attacker with a stolen session token from taking over the account.
```

