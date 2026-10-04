# Implementation Tasks: Open Finance API & Consent Management Hub

## Phase 1: TPP Trust & Edge Gateway
- [ ] **TSK-OBK-01**: Build Envoy ext_authz service validating eIDAS QWAC chains against the EU Trusted List with stapled OCSP and 1-hour revocation freshness.
- [ ] **TSK-OBK-02**: Implement QSealC JWS request-signature verification and PSD2 QCStatement role mapping (`PSP_AS`, `PSP_PI`, `PSP_AI`, `PSP_IC`) to API scopes.
- [ ] **TSK-OBK-03**: Build trust registry sync daemon for the EBA register and national registers with 15-minute client suspension on authorization withdrawal.
- [ ] **TSK-OBK-04**: Implement Redis-backed per-TPP and per-consent token buckets with `429` / `Retry-After` and customer-present traffic priority.

## Phase 2: FAPI 2.0 Authorization Server
- [ ] **TSK-OBK-05**: Implement PAR endpoint (single-use `request_uri`, 60 s expiry) with mandatory PKCE `S256` and RFC 9207 `iss` responses.
- [ ] **TSK-OBK-06**: Implement `private_key_jwt` / `tls_client_auth` client authentication and DPoP / mTLS sender-constrained access tokens (300 s lifetime).
- [ ] **TSK-OBK-07**: Build RFC 7591 / 7592 dynamic client registration with software statement validation and algorithm allow-list (`PS256`, `ES256`, `EdDSA`).

## Phase 3: Consent Engine & Customer Controls
- [ ] **TSK-OBK-08**: Implement the consent state machine and hash-chained `consent_events` ledger with append-only database grants.
- [ ] **TSK-OBK-09**: Enforce duration rules (180-day EU AIS re-SCA / re-confirmation, 1-year US Section 1033 authorization) and the 4-per-24h unattended AIS cap.
- [ ] **TSK-OBK-10**: Build the web and mobile permission dashboard with 2-click revocation and Kafka-driven cache invalidation in $\le 5\text{ s}$.
- [ ] **TSK-OBK-11**: Implement signed JWS consent receipts for grant, scope change, and revocation events.

## Phase 4: Payment Initiation, Dynamic Linking & Fraud Signals
- [ ] **TSK-OBK-12**: Implement SCA dynamic linking binding auth codes to `SHA-256(amount|currency|creditor|payment_id)` and rejecting any mismatch at execution.
- [ ] **TSK-OBK-13**: Build PIS endpoints (single, future-dated, standing order, bulk) with 24-hour `x-idempotency-key` replay protection.
- [ ] **TSK-OBK-14**: Implement ISO 20022 status mapping (`RCVD`, `ACTC`, `ACSC`, `RJCT`) and signed TPP webhooks within 2 s of core status change.
- [ ] **TSK-OBK-15**: Build fraud scoring service (device, payee novelty, velocity, TPP risk tier) with step-up SCA at score $\ge 0.85$.

## Phase 5: Regulatory KPI Reporting
- [ ] **TSK-OBK-16**: Build ClickHouse pipeline computing availability, p95/p99 latency, and proper-response rate for the dedicated interface vs. first-party channels.
- [ ] **TSK-OBK-17**: Generate quarterly EBA-format KPI statistics and monthly Section 1033 response-rate reports for compliance sign-off.

## Phase 6: Conformance, Load & Resilience Validation
- [ ] **TSK-OBK-18**: Pass the OpenID Foundation FAPI 2.0 Security Profile conformance suite for both DPoP and mTLS client variants.
- [ ] **TSK-OBK-19**: Load test 5,000 req/s sustained AIS traffic; verify p95 $\le 500\text{ ms}$, p99 $\le 1{,}000\text{ ms}$, and authorization overhead $\le 15\text{ ms}$ p99.
- [ ] **TSK-OBK-20**: Run 100,000 synthetic revocations under load; verify zero data returned after revocation beyond the 5 s propagation bound.
- [ ] **TSK-OBK-21**: Replay 50,000 tampered payment requests (amount, payee, idempotency); verify 100% rejection and zero duplicate executions.
