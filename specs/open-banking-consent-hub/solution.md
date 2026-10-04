# Solution Architecture: Open Finance API & Consent Management Hub

## 1. System Architecture Overview

The hub is a layered, zero-trust gateway that separates TPP trust, authorization, consent decisions, and resource access:
1. **Edge & Certificate Gateway (Envoy + Go ext_authz)**: Terminates mTLS, validates eIDAS QWACs against the EU Trusted List with stapled OCSP, verifies QSealC request signatures, and enforces per-TPP token-bucket rate limits backed by Redis.
2. **FAPI 2.0 Authorization Server (Go / Keycloak-compatible OIDC core)**: Handles PAR, PKCE, `private_key_jwt`, DPoP / mTLS token binding, and RFC 7591 / 7592 dynamic client registration; delegates SCA to the bank's IdP over OIDC CIBA or redirect.
3. **Consent Engine (Go / PostgreSQL / Kafka)**: Owns the consent state machine, duration limits, unattended-access counters, receipts, and the hash-chained consent ledger; publishes revocations on `consent.events` for sub-5 s cache invalidation.
4. **Resource API Layer (Kotlin / Spring Boot)**: Serves Berlin Group NextGenPSD2, UK Open Banking, and FDX-aligned AIS, PIS, and funds-confirmation endpoints over core banking adapters with field-level scope filtering.
5. **Risk, KPI & Registry Services (Python / ClickHouse)**: Fraud scoring for payment initiations, TPP register synchronization, and quarterly availability / latency statistics for regulators.

```
+-----------------------------------------------------------------------------------------+
|                                  THIRD-PARTY PROVIDERS                                  |
|        [EU AISP / PISP (QWAC + QSealC)]          [US Data Recipients (FDX / 1033)]      |
+-----------------------------------------------------------------------------------------+
                                          | mTLS / DPoP
                                          v
+-----------------------------------------------------------------------------------------+
|                         EDGE & CERTIFICATE GATEWAY (Envoy + Go)                         |
|  [QWAC Chain + OCSP] ---> [QSealC JWS Verify] ---> [Per-TPP Token Bucket (Redis)]       |
+-----------------------------------------------------------------------------------------+
                     |                                              |
                     v                                              v
+---------------------------------------+   +---------------------------------------------+
|     FAPI 2.0 AUTHORIZATION SERVER     |   |              RESOURCE API LAYER             |
|  - PAR / PKCE / private_key_jwt       |   |  - AIS: accounts, balances, transactions    |
|  - DPoP + mTLS bound tokens           |   |  - PIS: single, future, standing, bulk      |
|  - Dynamic Client Registration        |   |  - Funds confirmation (CBPII)               |
+---------------------------------------+   +---------------------------------------------+
                     |                                              |
                     v                                              v
+-----------------------------------------------------------------------------------------+
|                         CONSENT ENGINE (PostgreSQL + Kafka)                             |
|  [State Machine] -> [Duration / 4-per-day Limits] -> [Hash-Chained Ledger] -> [Receipts]|
+-----------------------------------------------------------------------------------------+
                     |                                              |
                     v                                              v
+---------------------------------------+   +---------------------------------------------+
|   BANK IdP & SCA (Dynamic Linking)    |   |   RISK, REGISTRY & KPI SERVICES             |
|  - Authenticator app push / FIDO2     |   |  - Fraud scoring / step-up                  |
|  - Permission dashboard (web/mobile)  |   |  - EBA register sync / KPI reports          |
+---------------------------------------+   +---------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Consent Registry (`consents`)
```sql
CREATE TABLE consents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tpp_client_id VARCHAR(128) NOT NULL,
    psu_id UUID NOT NULL,
    jurisdiction VARCHAR(8) NOT NULL CHECK (jurisdiction IN ('EU_PSD2', 'UK_OB', 'US_1033')),
    consent_type VARCHAR(16) NOT NULL CHECK (consent_type IN ('AIS', 'PIS', 'FUNDS_CONFIRM')),
    status VARCHAR(24) NOT NULL DEFAULT 'AWAITING_AUTH' CHECK (status IN ('AWAITING_AUTH', 'AUTHORISED', 'REJECTED', 'EXPIRED', 'REVOKED_BY_PSU', 'REVOKED_BY_TPP', 'REVOKED_BY_BANK', 'CONSUMED')),
    permissions TEXT[] NOT NULL CHECK (cardinality(permissions) > 0),
    account_ids UUID[] NOT NULL,
    payment_amount NUMERIC(18, 2) CHECK (payment_amount IS NULL OR payment_amount > 0),
    payment_currency CHAR(3),
    creditor_account VARCHAR(64),
    dynamic_link_hash CHAR(64), -- SHA-256(amount|currency|creditor|payment_id), PIS only
    authorised_at TIMESTAMPTZ,
    last_sca_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ NOT NULL,
    revoked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CHECK (consent_type <> 'PIS' OR (payment_amount IS NOT NULL AND dynamic_link_hash IS NOT NULL)),
    CHECK (jurisdiction <> 'US_1033' OR expires_at <= created_at + INTERVAL '365 days')
);
CREATE INDEX idx_consents_psu_active ON consents (psu_id) WHERE status = 'AUTHORISED';
```

### 2.2 Hash-Chained Consent Ledger (`consent_events`)
```sql
CREATE TABLE consent_events (
    seq BIGSERIAL PRIMARY KEY,
    consent_id UUID NOT NULL REFERENCES consents(id),
    event_type VARCHAR(32) NOT NULL CHECK (event_type IN ('CREATED', 'SCA_COMPLETED', 'SCOPE_REDUCED', 'REFRESHED', 'ACCESSED_UNATTENDED', 'REVOKED', 'EXPIRED', 'FRAUD_HOLD')),
    actor VARCHAR(16) NOT NULL CHECK (actor IN ('PSU', 'TPP', 'BANK', 'SYSTEM')),
    payload JSONB NOT NULL,
    prev_hash CHAR(64) NOT NULL,
    event_hash CHAR(64) NOT NULL UNIQUE, -- SHA-256(prev_hash || canonical(payload))
    occurred_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);
REVOKE UPDATE, DELETE ON consent_events FROM PUBLIC;
```

## 3. Consent Authorization Decision Algorithm
```go
var (
	ErrConsentDenied = errors.New("consent_denied")
	ErrRateLimited   = errors.New("rate_limited")
)

// Authorize runs on every resource request after the edge has verified the token's cnf binding.
func (e *ConsentEngine) Authorize(ctx context.Context, req ResourceRequest) (*Decision, error) {
	c, err := e.cache.GetOrLoad(ctx, req.ConsentID) // invalidated via consent.events within 5s
	if err != nil {
		return nil, err
	}
	now := e.clock.Now()

	if c.TPPClientID != req.ClientID || c.Status != StatusAuthorised || c.RevokedAt != nil {
		return deny(ErrConsentDenied, "consent not active for client")
	}
	if !now.Before(c.ExpiresAt) {
		return deny(ErrConsentDenied, "consent expired")
	}
	if c.Jurisdiction == EUPSD2 && c.Type == AIS && now.Sub(c.LastSCAAt) > 180*24*time.Hour {
		return deny(ErrConsentDenied, "renewed SCA or re-confirmation required (180-day limit)")
	}
	if !c.HasPermission(req.Permission) || !c.HasAccount(req.AccountID) {
		return deny(ErrConsentDenied, "scope or account not granted")
	}

	// EU AIS: unattended access (no PSU-IP-Address) capped at 4 per account per rolling 24h.
	if c.Jurisdiction == EUPSD2 && c.Type == AIS && req.PSUIPAddress == "" {
		key := fmt.Sprintf("unattended:%s:%s", c.ID, req.AccountID)
		n, err := e.redis.SlidingWindowIncr(ctx, key, 24*time.Hour)
		if err != nil {
			return nil, err
		}
		if n > 4 {
			return deny(ErrRateLimited, "unattended access limit reached")
		}
		e.ledger.Append(ctx, c.ID, EventAccessedUnattended, ActorTPP, req.AuditPayload())
	}

	return &Decision{Allow: true, FieldMask: c.FieldMaskFor(req.Permission)}, nil
}
```
