# Requirements Specification: Open Finance API & Consent Management Hub

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **Account Holder / PSU (ACT-PSU)**: Retail or business customer who grants, reviews, and revokes data access and payment consents and performs SCA.
- **Third-Party Provider (ACT-TPP)**: Licensed AISP, PISP, CBPII (EU) or authorized data recipient (US) consuming the APIs through a registered client.
- **Open Banking Operations Analyst (ACT-OPS)**: Onboards TPPs, monitors API health, handles TPP support tickets, and publishes KPI statistics.
- **Fraud & Risk Analyst (ACT-FRD)**: Reviews fraud signals on payment initiations and suspicious TPP access patterns; can block a TPP or consent.
- **Compliance Officer (ACT-CMP)**: Owns regulatory reporting to national competent authorities and the CFPB, and reviews consent audit evidence.
- **Trust Registry Sync Daemon (ACT-REG)**: Pulls TPP authorization status from the EBA register, national registers, and qualified trust service provider CRL/OCSP endpoints.

---

## 2. Functional Requirements

### 2.1 TPP Identity, Onboarding & Client Registration (FR-TPP)
- **FR-TPP-01 (eIDAS Certificate Validation)**: The system MUST validate EU TPP QWACs at TLS termination and QSealCs on signed requests: chain to a qualified trust service provider on the EU Trusted List, OCSP/CRL status no older than 1 hour, and PSD2 QCStatement roles (`PSP_AS`, `PSP_PI`, `PSP_AI`, `PSP_IC`) matching the requested API.
- **FR-TPP-02 (Dynamic Client Registration)**: The system MUST support RFC 7591 registration and RFC 7592 management using a signed software statement; registrations MUST pin `token_endpoint_auth_method` to `private_key_jwt` or `tls_client_auth` and reject any other method.
- **FR-TPP-03 (Regulatory Status Sync)**: The system MUST re-check every active TPP's authorization status against the EBA / national registers at least every 24 hours and suspend clients within 15 minutes of a detected withdrawal or passporting change.

### 2.2 FAPI 2.0 Authorization & Token Services (FR-AUTH)
- **FR-AUTH-01 (Pushed Authorization Requests)**: The authorization server MUST accept authorization requests only via PAR (RFC 9126); `request_uri` values MUST be single-use and expire in $\le 60\text{ s}$.
- **FR-AUTH-02 (PKCE & Response Integrity)**: The system MUST require PKCE with `S256`, return the `iss` parameter in authorization responses (RFC 9207), and issue authorization codes valid for $\le 60\text{ s}$ that are single-use.
- **FR-AUTH-03 (Sender-Constrained Tokens)**: Access tokens MUST be bound via mTLS (`cnf.x5t#S256`) or DPoP (`cnf.jkt`) and expire in $\le 300\text{ s}$; refresh tokens MUST be bound to the same client and never outlive their consent.
- **FR-AUTH-04 (Approved Algorithms)**: JWS signatures MUST use `PS256`, `ES256`, or `EdDSA`; `RS256` and `none` MUST be rejected.

### 2.3 Consent Lifecycle Management (FR-CON)
- **FR-CON-01 (Granular Scopes)**: Consents MUST be account-level and permission-level (e.g. `ReadBalances`, `ReadTransactionsDetail`, `ReadBeneficiaries`), and the PSU MUST be able to deselect any account or permission on the consent screen.
- **FR-CON-02 (Maximum Durations)**: AIS consents MUST require renewed SCA or TPP re-confirmation at most every 180 days per the amended RTS-SCA; US Section 1033 authorizations MUST expire after at most 1 year unless the consumer reauthorizes.
- **FR-CON-03 (Unattended Access Limit)**: For EU AIS, requests without the PSU present (no `PSU-IP-Address` header) MUST be capped at 4 per account per rolling 24 hours; customer-present requests MUST NOT count toward this cap.
- **FR-CON-04 (Permission Dashboard & Revocation)**: The PSU MUST see every active consent in online and mobile banking (TPP name, scopes, accounts, granted / expiry dates, last access) and revoke any consent in $\le 2$ clicks, with revocation propagated to all API nodes in $\le 5\text{ s}$.
- **FR-CON-05 (Consent Receipts)**: Each grant, change, and revocation MUST produce a signed (JWS) consent receipt emailed or pushed to the PSU and downloadable from the dashboard.

### 2.4 Payment Initiation & Dynamic Linking (FR-PAY)
- **FR-PAY-01 (Dynamic Linking)**: The SCA challenge MUST display the exact amount and payee, and the authentication code MUST be bound to $H(\text{amount} \,\|\, \text{currency} \,\|\, \text{creditor IBAN} \,\|\, \text{payment ID})$; any mismatch at execution MUST reject the payment.
- **FR-PAY-02 (Payment Types)**: The system MUST support single immediate (SEPA Instant, Faster Payments), future-dated, standing order, and bulk payments, plus funds confirmation for CBPIIs.
- **FR-PAY-03 (Status Reporting)**: Payment status MUST follow ISO 20022 codes (`RCVD`, `ACTC`, `ACSC`, `RJCT`) and be pushed to the TPP via signed webhooks within 2 s of a core banking status change.

### 2.5 Rate Limiting, Fraud Signals & KPI Reporting (FR-OPS)
- **FR-OPS-01 (Per-TPP Rate Limits)**: The system MUST apply token-bucket limits per TPP client and per consent (default 50 req/s burst, 20 req/s sustained per client) and return `429` with `Retry-After`; customer-present traffic MUST be prioritized ahead of batch traffic.
- **FR-OPS-02 (Fraud Signals)**: Each payment initiation MUST be scored using device, payee novelty, amount velocity, and TPP risk tier; scores $\ge 0.85$ MUST trigger step-up SCA or hold for analyst review.
- **FR-OPS-03 (KPI Statistics)**: The system MUST publish quarterly availability and response-time statistics for the dedicated interface alongside the bank's own customer channels, as required by EBA guidelines, and monthly Section 1033 response-rate figures.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Performance & Latency (NFR-PERF)
- **NFR-PERF-01 (API Latency)**: AIS read endpoints MUST respond in $\le 500\text{ ms}$ at p95 and $\le 1{,}000\text{ ms}$ at p99 at 5,000 req/s sustained.
- **NFR-PERF-02 (Authorization Overhead)**: Token introspection, certificate binding check, and consent evaluation combined MUST add $\le 15\text{ ms}$ p99.

### 3.2 Security & Privacy (NFR-SEC)
- **NFR-SEC-01 (FAPI Conformance)**: The authorization server MUST pass the OpenID Foundation FAPI 2.0 Security Profile conformance suite before every production release.
- **NFR-SEC-02 (Data Minimization)**: Responses MUST return only fields covered by consent scopes; PSU credentials MUST never be shared with or collected by TPPs.

### 3.3 Availability & Reliability (NFR-REL)
- **NFR-REL-01 (Interface Availability)**: The dedicated interface MUST reach $\ge 99.95\%$ monthly availability and a proper-response rate of $\ge 99.5\%$, measured separately from first-party channels.
- **NFR-REL-02 (Idempotent Payments)**: Payment initiation MUST honour `x-idempotency-key` for 24 hours; replays MUST return the original response and never create a second payment.

---

## 4. Payment Initiation Consent & SCA Flow

```
[PISP Client]                      [Consent Hub / AS]                    [Bank IdP & Core Banking]
     |                                     |                                        |
     |-- POST /par (private_key_jwt, ----->|                                        |
     |   PKCE S256, payment intent)        |-- Validate QWAC / QSealC / TPP status  |
     |<-- request_uri (60s) ---------------|                                        |
     |                                     |                                        |
     |-- Redirect PSU /authorize --------->|-- Fraud score (device, payee, amount)  |
     |                                     |-- SCA challenge: "EUR 250.00 to X" --->|
     |                                     |<-- Auth code bound H(amt|ccy|IBAN|id) -|
     |<-- code + iss (60s, single use) ----|                                        |
     |                                     |                                        |
     |-- POST /token (DPoP proof / mTLS) ->|-- Issue sender-constrained token       |
     |<-- access_token (cnf bound) --------|                                        |
     |                                     |                                        |
     |-- POST /payments (idempotency) ---->|-- Verify dynamic link hash ----------->|
     |                                     |                                        |-- Execute
     |<-- 201 ACTC / webhook ACSC ---------|<-- Status update ----------------------|
```
