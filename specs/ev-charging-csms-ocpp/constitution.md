# Constitution: EV Charging Station Management System (CSMS) & OCPP/OCPI Platform

## 1. Purpose & Mission
The EV Charging Station Management System (CSMS) is a mission-critical, multi-protocol backend for a Charge Point Operator (CPO) managing a fleet of 50,000 AC and DC chargers. It terminates persistent OCPP-J WebSocket connections (OCPP 2.1 / IEC 63584, OCPP 2.0.1, and legacy OCPP 1.6J), manages the charger device model and firmware lifecycle, provisions ISO 15118 Plug & Charge certificates, enforces site and grid capacity through smart charging, dispatches OCPP 2.1 bidirectional V2G and DER control signals, and settles every charging session through calibration-law-compliant signed metering, billing, and OCPI 2.2.1 roaming with eMobility Service Providers (eMSPs).

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: Site Power Never Exceeds the Grid Connection Limit
- At every instant $t$, the aggregate setpoint of all EVSEs at a site plus the measured non-EV building load must stay within the contracted grid connection capacity, minus a safety margin:
  $$\sum_{i \in \text{EVSEs}} P_i(t) + P_{\text{building}}(t) \le P_{\text{grid\_limit}}(t) \times (1 - m_{\text{safety}}), \quad m_{\text{safety}} \ge 0.05$$
- When a charger goes offline, its last-known allocation stays reserved until its `TxDefaultProfile` fallback limit is confirmed; capacity is never re-allocated on assumptions.

### Tenet 2: Signed Meter Values Are the Sole Billing Source of Truth
- Every billed kWh must trace back to a signed meter value (OCMF or OCPP `SignedMeterValueType`) whose signature verifies against the meter's registered public key. Unsigned or failed-signature readings may be shown to drivers but never invoiced in Eichrecht jurisdictions.
- Billed energy must equal $E_{\text{billed}} = E_{\text{stop}} - E_{\text{start}}$ from the signed register readings, with no rounding, interpolation, or estimation.

### Tenet 3: No Lost or Duplicated Transaction Events
- `TransactionEvent` messages are processed exactly-once by `(transactionId, seqNo)`. Offline-queued events (`offline = true`) replayed after reconnect must be accepted, ordered by `seqNo`, and reconciled; gaps are flagged, never silently ignored.

### Tenet 4: Security Profile Enforcement & Certificate Hygiene
- Public-internet chargers must use Security Profile 2 (TLS 1.2+ with HTTP Basic Auth) or Profile 3 (TLS 1.2+ mTLS client certificates); Profile 1 is allowed only on private APN / VPN networks. Downgrades of a charger's security profile are prohibited.
- Private keys for CSMS, V2G, and Sub-CA certificates live only in an HSM; expired or OCSP-revoked certificates are rejected immediately.

### Tenet 5: Protocol Fidelity & Backward Compatibility
- The CSMS speaks each OCPP version exactly as specified, with no proprietary message names and no schema shortcuts. All messages are validated against the official JSON schemas, and OCPP 1.6J chargers get equivalent business behavior through a version adapter.

## 3. Scope Boundaries

### What We Are Building
- Horizontally scaled OCPP-J WebSocket gateway (`ocpp1.6`, `ocpp2.0.1`, `ocpp2.1` subprotocols) with Security Profiles 1–3.
- Device model inventory, configuration, and firmware rollout orchestration.
- Smart charging engine (composite schedules, site load balancing, V2G/DER dispatch) and ISO 15118 Plug & Charge PKI integration.
- Transaction ledger, signed meter value verification, tariffing and billing, and OCPI 2.2.1 CPO interface.

### What We Are NOT Building
- We do not build charger firmware, embedded ISO 15118 EVCC/SECC stacks, or meter hardware.
- We do not operate as an eMSP (driver apps, driver contracts) or as a V2G Root CA / Contract Certificate Pool; we integrate with external PKI providers and roaming hubs.
