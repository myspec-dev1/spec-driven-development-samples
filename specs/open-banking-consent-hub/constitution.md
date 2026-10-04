# Constitution: Open Finance API & Consent Management Hub

## 1. Purpose & Mission
The Open Finance API & Consent Management Hub is the bank-side (ASPSP / data provider) gateway that exposes account information and payment initiation APIs to licensed third parties (AISPs, PISPs, and authorized data recipients). It enforces PSD2 RTS-SCA (Commission Delegated Regulation (EU) 2018/389) and the US CFPB Section 1033 Personal Financial Data Rights rule (12 CFR Part 1033), secures every call with the FAPI 2.0 Security Profile, and gives customers full, auditable control over who can see their data and move their money. The design is forward-compatible with the proposed EU PSD3 / Payment Services Regulation (PSR) permission dashboard and API performance obligations.

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: No Data Without a Live, Scoped Consent
- Every resource response must be covered by an active consent whose scopes, accounts, and validity window include the requested resource:
  $$\text{Allow}(r, t) \iff \exists\, c : c.\text{status} = \texttt{AUTHORISED} \land r \in c.\text{scopes} \times c.\text{accounts} \land t < c.\text{expires\_at} \land \neg c.\text{revoked}$$
- Revocation by the customer, the TPP, or the bank takes effect on the very next API call; cached authorization decisions must never outlive a revocation event by more than 5 seconds.

### Tenet 2: Sender-Constrained, Certificate-Bound Access Only
- Bearer tokens are prohibited. Every access token is bound to the client's mTLS certificate (RFC 8705) or DPoP key (RFC 9449), and the binding is verified on every request.
- EU TPPs must present a valid eIDAS QWAC (transport) and sign with a QSealC whose PSD2 roles (`PSP_AI`, `PSP_PI`, `PSP_IC`) and national competent authority registration match the requested operation.

### Tenet 3: Dynamic Linking Is Unbreakable
- A payment SCA authentication code is cryptographically bound to the exact amount and payee shown to the payer. Any change to amount, currency, or creditor account after SCA invalidates the authorization; the payment must never execute on a stale code.

### Tenet 4: Dedicated Interface Parity
- The third-party API must match the availability and performance of the bank's own customer channels (web and mobile). Throttling may protect platform stability but must never place customer-present TPP traffic below parity with first-party channels.

### Tenet 5: Immutable Consent Evidence
- Every consent creation, SCA event, scope change, refresh, and revocation is written to an append-only, hash-chained ledger and produces a customer-readable consent receipt. Records are retained for at least 5 years.

## 3. Scope Boundaries

### What We Are Building
- FAPI 2.0 authorization server (PAR, PKCE, private_key_jwt, mTLS / DPoP) with RFC 7591 / 7592 dynamic client registration.
- eIDAS QWAC/QSealC validation pipeline and TPP registry sync (EBA register, national registers, CFPB-recognized standard setters).
- Account information, payment initiation, and funds confirmation APIs (Berlin Group NextGenPSD2, UK Open Banking, and FDX-aligned profiles).
- Consent lifecycle engine, customer permission dashboard, consent receipts, per-TPP rate limiting, fraud signal scoring, and quarterly KPI reporting.

### What We Are NOT Building
- We do not act as a TPP or aggregator; we never consume other banks' open banking APIs.
- We do not settle, clear, or reconcile payments; executed payments are handed to the core banking and payment rails (reconciliation is owned by a separate engine).
- We do not run the customer's primary authentication stack; SCA is delegated to the bank's existing IdP and authenticator apps.
