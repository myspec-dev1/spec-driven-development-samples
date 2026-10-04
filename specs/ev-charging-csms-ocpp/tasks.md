# Implementation Tasks: EV Charging Station Management System (CSMS) & OCPP/OCPI Platform

## Phase 1: OCPP-J Gateway & Security Profiles
- [ ] **TSK-EVC-01**: Build Go WebSocket gateway with `ocpp2.1` / `ocpp2.0.1` / `ocpp1.6` subprotocol negotiation and CALL/CALLRESULT/CALLERROR correlation.
- [ ] **TSK-EVC-02**: Implement Security Profiles 1–3 (Basic Auth with Argon2id hashes, TLS 1.2+, mTLS with CN = `stationId`) and block profile downgrades.
- [ ] **TSK-EVC-03**: Integrate official OCPP JSON schemas for all three versions with `FormatViolation` / `ProtocolError` responses.
- [ ] **TSK-EVC-04**: Implement Redis station-affinity registry and NATS JetStream routing for CSMS-initiated CALLs across gateway nodes.

## Phase 2: Device Model, Firmware & PKI
- [ ] **TSK-EVC-05**: Implement `BootNotification`, `Heartbeat`, `StatusNotification` handling and `GetBaseReport` / `NotifyReport` device model storage.
- [ ] **TSK-EVC-06**: Build OCPP 1.6J protocol adapter mapping `StartTransaction` / `StopTransaction` / `GetConfiguration` to the 2.x domain model.
- [ ] **TSK-EVC-07**: Build staged `UpdateFirmware` campaign orchestrator (1% → 10% → 100%) with auto-halt at > 2% `InstallationFailed` / `InvalidSignature`.
- [ ] **TSK-EVC-08**: Implement HSM-backed `SignCertificate` / `CertificateSigned` flow and `Get15118EVCertificate` / `GetCertificateStatus` proxy with 24 h OCSP cache.

## Phase 3: Transactions & Signed Metering
- [ ] **TSK-EVC-09**: Implement `TransactionEvent` ledger with `(stationId, transactionId, seqNo)` idempotency, offline replay ordering, and gap detection.
- [ ] **TSK-EVC-10**: Build OCMF parser and ECDSA secp256r1 signature verifier; mark sessions with failed signatures as `INVALID_SIGNATURE` and block billing.
- [ ] **TSK-EVC-11**: Implement `Authorize` pipeline (local cache → OCPI real-time authorization) and `SendLocalList` synchronization.

## Phase 4: Smart Charging, V2G & DER
- [ ] **TSK-EVC-12**: Build site capacity tree model and weighted water-filling allocator enforcing $\sum P_i + P_{\text{building}} \le P_{\text{grid}}(1 - m)$.
- [ ] **TSK-EVC-13**: Implement `SetChargingProfile` / `ClearChargingProfile` dispatch with `TxDefaultProfile` offline fallback and `GetCompositeSchedule` verification.
- [ ] **TSK-EVC-14**: Implement `NotifyEVChargingNeeds` / `NotifyEVChargingSchedule` departure-time scheduling for ISO 15118 sessions.
- [ ] **TSK-EVC-15**: Implement OCPP 2.1 V2G `operationMode` setpoints and `SetDERControl` / `NotifyDERAlarm` handling for bidirectional EVSEs.

## Phase 5: Tariffs, Billing & OCPI 2.2.1 Roaming
- [ ] **TSK-EVC-16**: Implement OCPI 2.2.1 CPO modules (`credentials`, `locations`, `sessions`, `cdrs`, `tariffs`, `tokens`, `commands`) with Credentials token rotation.
- [ ] **TSK-EVC-17**: Build tariff engine (`ENERGY`, `TIME`, `PARKING_TIME`, `FLAT` with restrictions) and CDR generator embedding OCMF `signed_data`.

## Phase 6: Fleet-Scale Benchmarking & Validation
- [ ] **TSK-EVC-18**: Simulate 50,000 stations at 5,000 msg/s; verify p99 response latency $\le 500\text{ms}$ and reconnect storms of 10,000 stations/minute.
- [ ] **TSK-EVC-19**: Run a 24 h grid-limit fuzz test across 1,000 simulated sites; verify zero intervals where site power exceeds the grid connection limit.
- [ ] **TSK-EVC-20**: Replay 100,000 offline-queued `TransactionEvent` messages after a 7-day outage; verify zero lost or duplicated sessions and 100% OCMF verification pass rate on CDRs.
