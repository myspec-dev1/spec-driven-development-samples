# Solution Architecture: DSCSA Pharmaceutical Serialization & Traceability Platform

## 1. System Architecture Overview

The system is built for strict per-serial ordering, append-only evidence, and fast answers to regulators and trading partners:
1. **Serialization Service (Go / PostgreSQL)**: Owns GTIN master data and serial pools, allocates randomized serials to packaging lines, and converts between GS1 element strings, EPC URIs, and GS1 Digital Link.
2. **EPCIS 2.0 Capture & Query Gateway (Go / Kafka)**: Exposes the EPCIS 2.0 REST capture and query interface (JSON-LD and XML), validates CBV vocabulary, computes event hashes, and publishes to Kafka partitioned by `epc` so events for one serial are always processed in order.
3. **Serial Lifecycle & Aggregation Engine (Go / PostgreSQL + `ltree`)**: Applies events to the serial state machine, maintains the unit → case → pallet hierarchy, enforces the conservation invariant, and performs inference on intact containers.
4. **Trading Partner & TI/TS Exchange Service (TypeScript / NestJS)**: Stores partner profiles and exemption dates, runs Authorized Trading Partner checks against state license and FDA registration sources, and sends/receives TI/TS as EPCIS documents over AS2 or HTTPS.
5. **VRS Requester / Responder (Rust / Redis)**: Answers and issues GS1 Lightweight Messaging verification requests with a Redis hot cache of active serials and lookup-directory routing.
6. **Investigation & Compliance Workbench (TypeScript / React)**: Quarantine control, investigation case management, FDA Form 3911 prefill with 24-hour countdown, and a tracing-request responder.
7. **Evidence Archive (S3 Object Lock / Parquet + Trino)**: WORM storage of hash-chained events and documents for 6+ years, queryable for historical tracing requests.

```
+-----------------------------------------------------------------------------------------+
|                                      EVENT SOURCES                                      |
|                                                                                         |
|  [Packaging Lines (L3)]   [WMS / ERP Shipments]   [Partner EPCIS / TI-TS]   [EU EMVS]   |
+-----------------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                       EPCIS 2.0 CAPTURE & QUERY GATEWAY (Go)                            |
|                                                                                         |
|  [mTLS / OAuth2] ---> [CBV Schema Validator] ---> [Event Hash + De-dupe] ---> [ATP Gate]|
+-----------------------------------------------------------------------------------------+
                                          | (Kafka: epcis.events, partitioned by EPC)
                                          v
+-----------------------------------------------------------------------------------------+
|                         SERIAL LIFECYCLE & AGGREGATION ENGINE                           |
|                                                                                         |
|  [State Machine per Serial] ---> [Hierarchy (ltree)] ---> [Conservation + Inference]    |
+-----------------------------------------------------------------------------------------+
            |                             |                                 |
            v                             v                                 v
+-----------------------+   +-----------------------------+   +-------------------------------+
|  TI/TS EXCHANGE       |   |  VRS REQUESTER / RESPONDER  |   |  INVESTIGATION WORKBENCH      |
|  - ATP Verification   |   |  - Lookup Directory Routing |   |  - Quarantine / Case Mgmt     |
|  - Exemption Schedule |   |  - Redis Active-Serial Cache|   |  - Form 3911 (24h Countdown)  |
+-----------------------+   +-----------------------------+   |  - Tracing Request Responder  |
                                                              +-------------------------------+
                                          |
                                          v
                      [Evidence Archive: S3 Object Lock, 6-Year WORM]
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Serialized Unit Registry (`serialized_units`)
```sql
CREATE TABLE serialized_units (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    epc_uri VARCHAR(128) NOT NULL, -- urn:epc:id:sgtin:... or urn:epc:id:sscc:...
    id_type VARCHAR(8) NOT NULL CHECK (id_type IN ('SGTIN', 'SSCC')),
    gtin CHAR(14) CHECK (gtin ~ '^[0-9]{14}$'),
    serial_number VARCHAR(20),
    lot_number VARCHAR(20),
    expiry_date DATE,
    lifecycle_state VARCHAR(24) NOT NULL DEFAULT 'COMMISSIONED' CHECK (lifecycle_state IN (
        'COMMISSIONED', 'PACKED', 'IN_TRANSIT', 'RECEIVED', 'QUARANTINED',
        'DISPENSED', 'DESTROYED', 'STOLEN', 'SAMPLED', 'EXPORTED')),
    parent_id UUID REFERENCES serialized_units(id),
    hierarchy_path LTREE, -- e.g. pallet.case.unit, for subtree inference queries
    inference_broken BOOLEAN NOT NULL DEFAULT FALSE,
    current_owner_gln CHAR(13) NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    UNIQUE (tenant_id, epc_uri),
    CHECK (id_type = 'SSCC' OR (gtin IS NOT NULL AND serial_number IS NOT NULL
                                AND lot_number IS NOT NULL AND expiry_date IS NOT NULL)),
    CHECK (parent_id IS NULL OR parent_id <> id)
);
CREATE INDEX idx_units_hierarchy ON serialized_units USING GIST (hierarchy_path);
```

### 2.2 Immutable EPCIS Event Journal (`epcis_events`)
```sql
CREATE TABLE epcis_events (
    event_id VARCHAR(128) PRIMARY KEY, -- ni:///sha-256;... EPCIS 2.0 event hash
    tenant_id UUID NOT NULL,
    event_type VARCHAR(24) NOT NULL CHECK (event_type IN ('ObjectEvent', 'AggregationEvent', 'TransactionEvent')),
    action VARCHAR(8) NOT NULL CHECK (action IN ('ADD', 'OBSERVE', 'DELETE')),
    biz_step VARCHAR(32) NOT NULL CHECK (biz_step IN (
        'commissioning', 'packing', 'unpacking', 'shipping', 'receiving', 'decommissioning', 'destroying')),
    disposition VARCHAR(32),
    event_time TIMESTAMPTZ NOT NULL,
    event_time_zone_offset VARCHAR(6) NOT NULL CHECK (event_time_zone_offset ~ '^[+-][0-9]{2}:[0-9]{2}$'),
    record_time TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    parent_epc VARCHAR(128),
    epc_list TEXT[] NOT NULL,
    source_gln CHAR(13),
    destination_gln CHAR(13),
    biz_transaction_ref VARCHAR(128), -- PO / invoice / DESADV linking TI
    corrects_event_id VARCHAR(128) REFERENCES epcis_events(event_id),
    prev_hash CHAR(64) NOT NULL,
    chain_hash CHAR(64) NOT NULL, -- SHA-256(prev_hash || canonical event)
    payload JSONB NOT NULL,
    CHECK (event_type <> 'AggregationEvent' OR parent_epc IS NOT NULL)
);
```

## 3. Aggregation Conservation Check
```go
// ApplyPacking validates an EPCIS AggregationEvent (action=ADD, bizStep=packing)
// against the conservation invariant before committing it.
func (e *Engine) ApplyPacking(ctx context.Context, tx pgx.Tx, ev AggregationEvent) error {
	parent, err := e.repo.LockUnit(ctx, tx, ev.ParentEPC) // SELECT ... FOR UPDATE
	if err != nil {
		return fmt.Errorf("parent %s: %w", ev.ParentEPC, err)
	}
	for _, childEPC := range ev.ChildEPCs {
		child, err := e.repo.LockUnit(ctx, tx, childEPC)
		if err != nil {
			return ErrSuspect{EPC: childEPC, Reason: "child never commissioned"}
		}
		// Disjointness: a child may belong to exactly one parent.
		if child.ParentID != nil && *child.ParentID != parent.ID {
			return ErrSuspect{EPC: childEPC, Reason: "child already aggregated to another parent"}
		}
		if child.State.IsTerminal() || child.State == StateQuarantined {
			return ErrSuspect{EPC: childEPC, Reason: "child not in saleable state: " + string(child.State)}
		}
		if err := e.repo.Attach(ctx, tx, child.ID, parent.ID, parent.Path); err != nil {
			return err
		}
	}

	// Conservation: expanded unit set of the parent == union of children's unit sets.
	expected, err := e.repo.CountLeafUnitsUnder(ctx, tx, ev.ChildEPCs)
	if err != nil {
		return err
	}
	actual, err := e.repo.CountLeafUnitsByPath(ctx, tx, parent.Path) // ltree: path <@ parent.Path
	if err != nil {
		return err
	}
	if actual != parent.LeafCountBefore+expected {
		return ErrConservation{Parent: parent.EPC, Expected: parent.LeafCountBefore + expected, Actual: actual}
	}
	return e.journal.Append(ctx, tx, ev) // hash-chained insert into epcis_events
}
```
