# 13 — Authentication and Security

---

## 1. Authentication

### 1.1 Mechanism
- **JWT (JSON Web Tokens)** for all API access
- Access token: short-lived (15 minutes recommended)
- Refresh token: longer-lived (7 days), stored securely (HTTP-only cookie or secure storage)
- All tokens include: `user_id`, `client_id`, `role`, `exp`

### 1.2 Login Flow
```
1. POST /auth/login (email + password)
2. Server verifies credentials against accounts.users
3. Issues access_token + refresh_token
4. Client stores refresh_token in HTTP-only cookie; access_token in memory
5. Access_token used as Bearer on all API calls
6. On 401: use refresh_token to get new access_token
7. On refresh failure: redirect to login
```

### 1.3 Password Security
- Passwords hashed with bcrypt (cost factor ≥12)
- No plain-text passwords stored anywhere
- Password reset via email link (time-limited, single-use token)

### 1.4 Session Management
- Each refresh token is single-use (rotated on use)
- All sessions listed and revocable by admin
- Session logout invalidates refresh token (stored in `accounts.sessions`)

### 1.5 Big Screen / Kiosk Authentication
- Big screen runs at a device-pinned URL with a long-lived device token
- Device token is not a user session; has only big-screen-read permissions
- Token managed by War Room Admin; can be revoked

### 1.6 CM View Authentication
- Dedicated simplified login (or PIN) for CM View mode
- Access only to CM summary and big screen (read-only, no corrections)
- Session timeout: short (30 minutes of inactivity)

---

## 2. Authorisation (Role-Based Access Control)

### 2.1 Role Hierarchy
```
system_admin > war_room_admin > war_room_lead > team_member > cm_view
```

### 2.2 Enforcement
- Role enforced at the API layer on every request
- `client_id` from JWT always used for data scoping — never from request body
- Cross-client data access is impossible by design; tested explicitly

### 2.3 Sensitive Actions Requiring Elevated Role
| Action | Minimum Role |
|--------|-------------|
| Campaign attribution confirmation | war_room_lead |
| Ally-blame review (CM summary) | war_room_lead |
| Second-review of biased corrections | war_room_lead |
| Configuration changes (tiers, watch terms, thresholds) | war_room_admin |
| Audit log access | war_room_admin |
| User management | system_admin |

---

## 3. Data Security

### 3.1 Client Isolation
- Every database row has a `client_id` column
- All queries include `WHERE client_id = ?` — enforced at ORM layer
- Row-level security (PostgreSQL RLS) as additional layer [PROPOSED]
- Cross-client isolation tested in automated test suite with cross-client attack scenarios

### 3.2 Data in Transit
- HTTPS/TLS 1.2+ on all connections
- WSS (WebSocket Secure) for real-time connections
- TLS certificates: auto-renewed (Let's Encrypt or ACM)

### 3.3 Data at Rest
- Database: encrypted at rest (cloud provider managed encryption)
- Object storage (S3): server-side encryption enabled
- Secrets (API keys, credentials): AWS Secrets Manager or equivalent; never in code or logs

### 3.4 Personal Data — DPDP Act 2023 Compliance
- India's DPDP Act 2023 and Rules apply to all personal data stored
- **Data residency:** India region only (AWS ap-south-1 or equivalent)
- Personal data stored: user accounts (name, email, WhatsApp number), X author profiles (handle, follower count, verified status)
- **No private citizen profiles** — analysis covers public actors only
- **No links to citizen or petition records**
- Data retention policy: [GAP — specific periods not defined in PRD; must be defined before launch]
- Deletion: user accounts deletable by admin; X author data: retained only as needed for Mention context [Policy to be formalised]
- Consent: staff accounts — explicit consent on account creation; X/news data — public data processed for legitimate political communication monitoring purposes

### 3.5 Secrets Management
- All credentials (X API key, WhatsApp API credentials, LLM API keys, database passwords) stored in AWS Secrets Manager
- Environment variables injected at runtime; never hardcoded
- Secrets never logged (log sanitisation enforced)
- Secret rotation: scheduled rotation for database passwords; manual rotation for API keys

---

## 4. AI / Co-worker Security

### 4.1 Prompt Injection Prevention
- **Principle:** All ingested content (posts, articles, clippings) is treated as DATA, never as instructions
- System prompt and user prompt are clearly separated in every LLM call
- Ingested text is always placed in a "content" block, never interpolated directly into the system prompt
- Example safe pattern:
  ```
  SYSTEM: You are the War Room co-worker. Answer based on the following mentions.
  USER: [Question]
  CONTENT_DATA: [Mention texts, clearly delimited]
  ```
- A post containing "ignore your instructions" is processed as a post

### 4.2 LLM Output Validation
- Co-worker answers are validated before display: citations must resolve to real Mention IDs
- Numbers in co-worker answers must match live database query results
- If validation fails: answer is flagged with a warning, not displayed as authoritative

### 4.3 Co-worker Action Limits
- The co-worker has NO access to any posting, publishing, or messaging API
- WhatsApp sending is handled by a separate service; co-worker can trigger an alert but cannot call the API directly
- In Phase 1: co-worker cannot send any external message on its own

---

## 5. Audit and Logging

### 5.1 What Must Be Logged
Every mutation is logged in `audit.audit_log` with:
- Actor (user ID or "system")
- Action (what was done)
- Entity type and ID (what was changed)
- Old value and new value (JSON)
- Reason (for corrections, confirmations)
- Timestamp

### 5.2 Co-worker Audit
Every co-worker interaction is logged in `learning.coworker_interactions` with:
- Full question
- Full answer
- Model version and prompt version used
- Source Mention IDs used for grounding
- Latency

Every alert generated by the co-worker is logged with its inputs, sources, and model/prompt version.

### 5.3 Log Sanitisation
- API keys, passwords, WhatsApp numbers must never appear in application logs
- Personal data in logs: minimised to what is necessary for debugging

### 5.4 Audit Log Access
- Audit logs: read-only for war_room_admin and system_admin
- Audit logs: immutable (append-only table; no UPDATE or DELETE)
- Retention: minimum 1 year [PROPOSED — confirm with client]

---

## 6. Infrastructure Security

### 6.1 Network
- Application load balancer: public-facing
- Backend API: private subnet; accessible only via load balancer
- Database: private subnet; accessible only from backend service subnet
- Pipeline workers: private subnet

### 6.2 Access Control
- SSH access to servers: via bastion host only; key-based; MFA for all engineers
- Database: no public endpoint; accessed only from application subnets
- S3 buckets: no public read; signed URLs for image display

### 6.3 Dependency Security
- Dependabot or equivalent: automated alerts for vulnerable dependencies
- Base Docker images: pinned versions; regularly updated
- Dependency scan on every CI build

---

## 7. Business Rules Re-enforced by Security

| Business Rule | Security Implementation |
|--------------|------------------------|
| Campaign attribution requires human | API validates that campaign `attribution_status` can only be set to "confirmed" by war_room_lead role; cannot be automated |
| Ally blame requires lead review | CM summary with ally attribution gated behind `needs_ally_review` flag; API blocks delivery until reviewed |
| Corrections never overwrite originals | Label records are INSERT-only; corrections are new records; UPDATE on `mention_labels` is not permitted at DB level |
| Co-worker never publishes | No WhatsApp API or posting API credentials accessible to LLM service |
| Presentation-safe mode | Enforced server-side: private handles filtered before API response when mode is active |
