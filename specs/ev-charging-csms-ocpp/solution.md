# Solution Architecture: EV Charging Station Management System (CSMS) & OCPP/OCPI Platform

## 1. System Architecture Overview

The system separates stateful WebSocket termination from stateless business services so that 50,000 long-lived connections can scale independently of transaction and smart-charging logic:
1. **OCPP Edge Gateway (Go / gorilla-websocket / Envoy L4 TLS passthrough)**: Terminates TLS 1.2+/mTLS, negotiates `ocpp1.6` / `ocpp2.0.1` / `ocpp2.1`, validates JSON schemas, and publishes normalized messages to NATS JetStream. Station-to-node affinity is kept in Redis so any service can route CSMS-initiated CALLs.
2. **Protocol Adapter & Device Service (Go / PostgreSQL)**: Translates OCPP 1.6J messages into the 2.x domain model, stores the device model (Component/Variable/Attribute) tree, and orchestrates `SetVariables` and staged `UpdateFirmware` campaigns.
3. **Transaction & Metering Ledger (Go / PostgreSQL + TimescaleDB)**: Applies `TransactionEvent` messages idempotently by `seqNo`, stores meter value hypertables, and verifies OCMF / signed meter value signatures before sessions become billable.
4. **Smart Charging & Grid Engine (Go / Redis)**: Builds per-site capacity trees (grid connection → transformer → feeder → EVSE), runs the allocation loop, and emits `SetChargingProfile` and OCPP 2.1 `SetDERControl` commands.
5. **PKI & Plug & Charge Service (Go / AWS CloudHSM / PKCS#11)**: Signs station and V2G CSRs, proxies `Get15118EVCertificate` to the Contract Certificate Pool, and caches OCSP responses.
6. **Billing & OCPI Hub Connector (TypeScript / NestJS / PostgreSQL)**: Prices sessions with OCPI 2.2.1 tariffs, generates CDRs with embedded `signed_data`, and exposes the CPO OCPI endpoints to eMSPs and roaming hubs.

```
+-------------------------------------------------------------------------------------------+
|                         CHARGING STATION FLEET (50,000 chargers)                          |
|   [OCPP 1.6J AC Wallboxes]     [OCPP 2.0.1 DC Fast Chargers]     [OCPP 2.1 V2G Chargers]  |
+-------------------------------------------------------------------------------------------+
                                            | wss:// (Security Profile 1/2/3)
                                            v
+-------------------------------------------------------------------------------------------+
|                         OCPP EDGE GATEWAY (Go, 20 nodes x 5k conns)                       |
|  [TLS / mTLS Termination] -> [Subprotocol Negotiation] -> [JSON Schema Validation]        |
|  [Station Affinity Registry (Redis)]          [CALL / CALLRESULT Correlation Table]       |
+-------------------------------------------------------------------------------------------+
                                            | NATS JetStream (ocpp.in.* / ocpp.out.{node})
           +------------------------+-------+----------------+--------------------------+
           |                        |                        |                          |
           v                        v                        v                          v
+--------------------+  +-----------------------+  +----------------------+  +--------------------+
| DEVICE & FIRMWARE  |  | TRANSACTION & METERING|  | SMART CHARGING & GRID|  | PKI & PLUG&CHARGE  |
| - Device Model DB  |  | - seqNo Idempotency   |  | - Site Capacity Tree |  | - HSM Sub-CA       |
| - Firmware Waves   |  | - OCMF Verification   |  | - Composite Schedule |  | - OCSP Cache       |
| - 1.6J Adapter     |  | - TimescaleDB MV      |  | - V2G / DER Dispatch |  | - CCP Proxy (EXI)  |
+--------------------+  +-----------------------+  +----------------------+  +--------------------+
                                    |
                                    v
+-------------------------------------------------------------------------------------------+
|                    BILLING & OCPI 2.2.1 CONNECTOR (NestJS / PostgreSQL)                   |
|  [Tariff Engine] -> [CDR Generator + signed_data] -> [eMSP / Roaming Hub Push (OCPI)]     |
+-------------------------------------------------------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Charging Transactions Ledger (`charging_transactions`)
```sql
CREATE TABLE charging_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    station_id VARCHAR(48) NOT NULL,
    evse_id INTEGER NOT NULL CHECK (evse_id > 0),
    connector_id INTEGER NOT NULL CHECK (connector_id > 0),
    ocpp_transaction_id VARCHAR(36) NOT NULL,
    ocpp_version VARCHAR(8) NOT NULL CHECK (ocpp_version IN ('1.6J', '2.0.1', '2.1')),
    id_token VARCHAR(255) NOT NULL,
    id_token_type VARCHAR(20) NOT NULL CHECK (id_token_type IN ('ISO14443', 'ISO15693', 'eMAID', 'Central', 'KeyCode', 'Local', 'MacAddress', 'NoAuthorization')),
    emsp_party_id VARCHAR(5), -- OCPI country_code + party_id, e.g. 'DEABC'
    state VARCHAR(16) NOT NULL DEFAULT 'STARTED' CHECK (state IN ('STARTED', 'UPDATED', 'ENDED', 'BILLED', 'INVALID_SIGNATURE')),
    last_seq_no INTEGER NOT NULL DEFAULT 0 CHECK (last_seq_no >= 0),
    started_at TIMESTAMPTZ NOT NULL,
    ended_at TIMESTAMPTZ,
    meter_start_wh NUMERIC(14, 3) NOT NULL CHECK (meter_start_wh >= 0),
    meter_stop_wh NUMERIC(14, 3),
    energy_discharged_wh NUMERIC(14, 3) NOT NULL DEFAULT 0 CHECK (energy_discharged_wh >= 0), -- V2G export
    stopped_reason VARCHAR(32),
    total_cost NUMERIC(12, 4),
    currency VARCHAR(3),
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    UNIQUE (station_id, ocpp_transaction_id),
    CHECK (meter_stop_wh IS NULL OR meter_stop_wh >= meter_start_wh),
    CHECK (ended_at IS NULL OR ended_at >= started_at)
);
```

### 2.2 Signed Meter Values Archive (`signed_meter_values`)
```sql
CREATE TABLE signed_meter_values (
    id BIGSERIAL PRIMARY KEY,
    transaction_id UUID NOT NULL REFERENCES charging_transactions(id),
    seq_no INTEGER NOT NULL CHECK (seq_no >= 0),
    reading_context VARCHAR(24) NOT NULL CHECK (reading_context IN ('Transaction.Begin', 'Sample.Periodic', 'Sample.Clock', 'Transaction.End')),
    measurand VARCHAR(48) NOT NULL DEFAULT 'Energy.Active.Import.Register',
    value_wh NUMERIC(14, 3) NOT NULL,
    sampled_at TIMESTAMPTZ NOT NULL,
    signed_meter_data TEXT NOT NULL, -- raw OCMF: 'OCMF|{payload}|{signature}'
    signing_method VARCHAR(50) NOT NULL, -- e.g. 'ECDSA-secp256r1-SHA256'
    encoding_method VARCHAR(50) NOT NULL, -- e.g. 'OCMF'
    meter_public_key TEXT NOT NULL,
    signature_valid BOOLEAN NOT NULL,
    verified_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    UNIQUE (transaction_id, seq_no, reading_context, measurand)
);
```

## 3. Site Capacity Allocation & Charging Profile Dispatch
```go
// AllocateSite splits available site power across active EVSEs so that
// sum(P_i) + P_building <= P_grid * (1 - margin), honoring each EVSE's
// min/max limits, priority weight, and V2G discharge setpoints.
func AllocateSite(site Site, evses []EVSEDemand, now time.Time) ([]SetChargingProfileRequest, error) {
	available := site.GridLimitW*(1-site.SafetyMargin) - site.BuildingLoadW(now)

	// V2G discharge (negative setpoints) frees capacity for other EVSEs.
	for _, e := range evses {
		if e.Bidirectional && e.DischargeSetpointW < 0 {
			available -= e.DischargeSetpointW
		}
	}

	// Guarantee each charging EVSE its minimum (e.g. 6 A x 230 V x phases) first.
	alloc := make(map[int]float64, len(evses))
	var chargers []EVSEDemand
	for _, e := range evses {
		if e.DischargeSetpointW < 0 {
			continue
		}
		alloc[e.ID] = e.MinW
		available -= e.MinW
		chargers = append(chargers, e)
	}
	if available < 0 {
		return nil, ErrSiteOverCommitted // pause lowest-priority EVSEs upstream
	}

	// Water-filling: share the remainder by weight, capped at each EVSE's max.
	for available > 1 && len(chargers) > 0 {
		var totalWeight float64
		for _, e := range chargers {
			totalWeight += e.Weight
		}
		next := chargers[:0]
		distributed := 0.0
		for _, e := range chargers {
			share := available * e.Weight / totalWeight
			if alloc[e.ID]+share >= e.MaxW {
				distributed += e.MaxW - alloc[e.ID]
				alloc[e.ID] = e.MaxW
			} else {
				alloc[e.ID] += share
				distributed += share
				next = append(next, e)
			}
		}
		available -= distributed
		if len(next) == len(chargers) {
			break // nobody capped: remainder fully distributed
		}
		chargers = next
	}

	reqs := make([]SetChargingProfileRequest, 0, len(evses))
	for _, e := range evses {
		limit, mode := alloc[e.ID], OperationModeChargingOnly
		if e.DischargeSetpointW < 0 {
			limit, mode = e.DischargeSetpointW, OperationModeCentralSetpoint // OCPP 2.1 only
		}
		reqs = append(reqs, BuildTxProfile(e, limit, mode, now)) // ChargingRateUnit: W
	}
	return reqs, nil
}
```
