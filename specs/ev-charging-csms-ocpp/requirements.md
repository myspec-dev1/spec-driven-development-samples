# Requirements Specification: EV Charging Station Management System (CSMS) & OCPP/OCPI Platform

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **Charging Station (ACT-CS)**: OCPP 1.6J / 2.0.1 / 2.1 charger that keeps a persistent WebSocket connection, reports status, transactions, and meter values, and executes CSMS commands.
- **CPO Network Operations Engineer (ACT-NOC)**: Monitors fleet health, triages faults, schedules firmware rollouts, and issues remote commands (Reset, UnlockConnector, TriggerMessage).
- **Energy & Grid Manager (ACT-GRD)**: Configures site grid connection limits, tariff-driven load shifting, and V2G/DER participation for DSO/TSO flexibility programs.
- **eMobility Service Provider (ACT-EMSP)**: Roaming partner that exchanges Tokens, Sessions, CDRs, and Tariffs with the CPO over OCPI 2.2.1, directly or via a roaming hub.
- **EV Driver (ACT-DRV)**: Starts sessions with RFID, app, ad-hoc payment, or ISO 15118 Plug & Charge, and verifies signed meter data with transparency software.
- **V2G PKI Provider (ACT-PKI)**: External V2G Root CA, Contract Certificate Pool, and OCSP responder (e.g. Hubject) that issue and validate ISO 15118 certificates.

---

## 2. Functional Requirements

### 2.1 OCPP Connectivity & Security (FR-CON)
- **FR-CON-01 (Multi-Version OCPP-J Endpoint)**: The system MUST accept WebSocket upgrades at `wss://<host>/ocpp/{stationId}` and negotiate the `Sec-WebSocket-Protocol` subprotocol `ocpp2.1`, `ocpp2.0.1`, or `ocpp1.6`, preferring the highest version the station offers.
- **FR-CON-02 (Security Profiles 1–3)**: The system MUST support Profile 1 (HTTP Basic Auth, private networks only), Profile 2 (TLS 1.2+ server auth + Basic Auth with a 16–40 character password), and Profile 3 (TLS 1.2+ mutual auth with a client certificate whose CN equals the `stationId`).
- **FR-CON-03 (Boot & Heartbeat)**: On `BootNotification`, the system MUST respond `Accepted`, `Pending`, or `Rejected`, with an `interval` of 300 s by default. A station with no message for $2 \times$ its heartbeat interval MUST be marked `Offline`.
- **FR-CON-04 (Schema Validation & RPC Framing)**: Every CALL (`[2, ...]`), CALLRESULT (`[3, ...]`), and CALLERROR (`[4, ...]`) frame MUST be validated against the official OCPP JSON schemas for the negotiated version. Invalid payloads MUST return `FormatViolation` or `ProtocolError`. OCPP 2.1 `SEND` (`[6, ...]`) and `CALLRESULTERROR` (`[5, ...]`) frames MUST be supported.
- **FR-CON-05 (Certificate Lifecycle)**: The system MUST handle `SignCertificate` CSRs (ChargingStationCertificate, V2GCertificate), sign them through the HSM-backed Sub-CA, and return them with `CertificateSigned` at least 30 days before expiry.

### 2.2 Device Model & Firmware Management (FR-DEV)
- **FR-DEV-01 (Device Model Inventory)**: The system MUST call `GetBaseReport` (`FullInventory`) after the first boot and store the `NotifyReport` Component/Variable/Attribute tree for every station. For OCPP 1.6J, `GetConfiguration` keys MUST be mapped to the same model.
- **FR-DEV-02 (Bulk Configuration)**: The system MUST apply `SetVariables` (2.x) or `ChangeConfiguration` (1.6J) to cohorts of up to 50,000 stations, tracking `Accepted`, `RebootRequired`, and `Rejected` per station.
- **FR-DEV-03 (Staged Firmware Rollout)**: The system MUST send signed-firmware `UpdateFirmware` requests in canary waves (1% → 10% → 100%). It MUST track `FirmwareStatusNotification` states and stop the rollout automatically if `InstallationFailed` or `InvalidSignature` exceeds 2% of a wave.
- **FR-DEV-04 (Security Event Monitoring)**: The system MUST store every `SecurityEventNotification` (e.g. `InvalidFirmwareSignature`, `TamperDetectionActivated`) and alert the NOC within 60 s.

### 2.3 Transactions, Authorization & Plug & Charge (FR-TRX)
- **FR-TRX-01 (Transaction Lifecycle)**: The system MUST process `TransactionEvent` `Started`/`Updated`/`Ended` (2.x) and `StartTransaction`/`StopTransaction`/`MeterValues` (1.6J) into one normalized transaction ledger, idempotent on `(stationId, transactionId, seqNo)`.
- **FR-TRX-02 (Authorization)**: The system MUST answer `Authorize` within 2 s using the local token cache, then OCPI real-time authorization (`POST /tokens/{uid}/authorize`) for roaming tokens. It MUST also manage station Local Authorization Lists via `SendLocalList`.
- **FR-TRX-03 (ISO 15118 Plug & Charge)**: The system MUST relay `Get15118EVCertificate` EXI requests (ISO 15118-2 and -20 schemas) to the Contract Certificate Pool, and validate contract certificate chains with `GetCertificateStatus` OCSP checks (responses cached ≤ 24 h).
- **FR-TRX-04 (Signed Meter Values)**: The system MUST store the raw `signedMeterData`, `signingMethod`, `encodingMethod`, and `publicKey` of every signed reading. It MUST verify OCMF payloads (ECDSA secp256r1 / SHA-256) against the meter's registered public key before billing.

### 2.4 Smart Charging & Grid Capacity (FR-SMC)
- **FR-SMC-01 (Charging Profile Stack)**: The system MUST manage `ChargingStationMaxProfile`, `TxDefaultProfile`, and `TxProfile` using `SetChargingProfile`, `ClearChargingProfile`, and `GetChargingProfiles` with stack levels 0–N. Results MUST be verified with `GetCompositeSchedule`.
- **FR-SMC-02 (Site Load Balancing)**: The system MUST recompute site allocations within 5 s of any session start/stop or grid limit change, so that:
  $$\sum_{i=1}^{n} P_i(t) \le P_{\text{grid}}(t) - P_{\text{building}}(t) - P_{\text{margin}}$$
- **FR-SMC-03 (ISO 15118 Charging Needs)**: The system MUST use `NotifyEVChargingNeeds` (departure time, energy amount) to build per-EV `TxProfile` schedules, and accept EV renegotiation through `NotifyEVChargingSchedule`.
- **FR-SMC-04 (OCPP 2.1 V2G & DER Control)**: For OCPP 2.1 bidirectional EVSEs, the system MUST dispatch discharge setpoints (negative power) using charging schedule periods with `operationMode` (`CentralSetpoint`, `ExternalLimits`, `CentralFrequency`, `Idle`). It MUST also manage grid-code curves through `SetDERControl` / `ClearDERControl` and process `NotifyDERAlarm`.
- **FR-SMC-05 (Offline Fallback)**: Every station MUST hold a `TxDefaultProfile` fallback limit equal to its fair share of $P_{\text{grid}} / n_{\text{EVSE}}$, so that the site stays within the grid limit even when the CSMS connection is lost.

### 2.5 Tariffs, Billing & OCPI Roaming (FR-ROM)
- **FR-ROM-01 (OCPI 2.2.1 CPO Modules)**: The system MUST implement the `credentials`, `locations`, `sessions`, `cdrs`, `tariffs`, `tokens`, `commands`, and `chargingprofiles` modules (CPO role), including the Credentials handshake and token rotation.
- **FR-ROM-02 (Session & CDR Push)**: The system MUST push `Session` updates to the eMSP at least every 15 min while a session is active. It MUST push a final `CDR` within 15 min of `TransactionEvent(Ended)`, including the `signed_data` (OCMF) block for Eichrecht verification.
- **FR-ROM-03 (Tariff Calculation)**: The system MUST price sessions with OCPI tariff elements (`ENERGY` per kWh, `TIME` per hour, `PARKING_TIME`, `FLAT`) and restrictions (time of day, `min_kwh`, `max_power`). Amounts MUST be calculated in `NUMERIC` with VAT applied per country.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Performance & Scale (NFR-PERF)
- **NFR-PERF-01 (Connection Scale)**: The gateway MUST hold 50,000 concurrent WebSocket connections with headroom to 100,000, and accept reconnect storms of 10,000 stations/minute after a regional outage.
- **NFR-PERF-02 (Message Latency)**: `Authorize`, `BootNotification`, and `TransactionEvent` responses MUST be sent in p99 $\le 500\text{ms}$ at a sustained 5,000 messages/s.

### 3.2 Security & Compliance (NFR-SEC)
- **NFR-SEC-01 (Key Custody)**: Sub-CA and CSMS TLS private keys MUST be stored in a FIPS 140-2 Level 3 HSM. Basic Auth passwords MUST be stored as Argon2id hashes and rotated with `SetVariables` (`SecurityCtrlr.BasicAuthPassword`).
- **NFR-SEC-02 (Eichrecht Audit Trail)**: Signed meter records and CDRs MUST be retained unaltered for at least 10 years and be verifiable offline by S.A.F.E. transparency software.

### 3.3 Availability & Reliability (NFR-REL)
- **NFR-REL-01 (Gateway Availability)**: The OCPP gateway MUST provide 99.95% monthly availability, and a single gateway node failure MUST NOT drop more than 2% of connections.
- **NFR-REL-02 (Offline Replay)**: Stations offline for up to 7 days MUST be able to replay queued `TransactionEvent` messages with zero lost or duplicated sessions.

---

## 4. Plug & Charge Session & Smart Charging Flow

```
[EV (ISO 15118-2/-20)]      [Charging Station]            [CSMS]                   [PKI / eMSP]
        |                          |                          |                          |
        |-- Contract Cert -------->|                          |                          |
        |                          |-- Authorize (eMAID + --->|                          |
        |                          |   iso15118CertHashData)  |-- OCSP / Token check --->|
        |                          |<- Authorize(Accepted) ---|<- Valid -----------------|
        |-- ChargeParameter ------>|                          |                          |
        |   (EAmount, DepartureT)  |-- NotifyEVChargingNeeds->|                          |
        |                          |                          |-- Site Load Balancer ----+
        |                          |<- SetChargingProfile ----|   (sum P_i <= P_grid)    |
        |                          |   (TxProfile, 22 kW)     |                          |
        |                          |-- TransactionEvent ----->|                          |
        |                          |   (Started, seqNo=0)     |-- OCPI PUT Session ----->|
        |<= Energy Transfer ======>|-- TransactionEvent ----->|                          |
        |                          |   (Updated, signed MV)   |-- OCPI PATCH Session --->|
        |-- SessionStop ---------->|-- TransactionEvent ----->|                          |
        |                          |   (Ended, OCMF signed)   |-- Verify OCMF signature  |
        |                          |                          |-- Tariff -> CDR -------->|
```
