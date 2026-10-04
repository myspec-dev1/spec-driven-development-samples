# Solution Architecture: EU Digital Battery Passport Platform

## 1. System Architecture Overview

The system is architected for long-lived resolvability, cryptographic provenance, and strict tier-based disclosure:
1. **GS1 Digital Link Resolver (Go / Envoy / CloudFront edge cache)**: Parses `/01/{gtin}/21/{serial}` URIs, authenticates the caller tier, negotiates content type, and serves tier-filtered passport views from a signed snapshot cache.
2. **Passport Registry Service (TypeScript / NestJS / PostgreSQL 16)**: Mints identifiers, stores versioned passport attribute sets, enforces the lifecycle state machine, and handles passport succession on repurposing and remanufacturing.
3. **Credential Verification Service (Rust / `ssi` crate)**: Verifies W3C Verifiable Credentials 2.0 proofs, resolves `did:web` / `did:key` issuers, and polls Bitstring Status Lists for revocation.
4. **Telemetry Ingestion Pipeline (EMQX MQTT 5 / Apache Kafka / TimescaleDB)**: Receives signed BMS readings, runs SoH plausibility filtering, and publishes a daily dynamic-attribute digest to the registry.
5. **Federation & Backup Replicator (Go / S3 Object Lock / IPFS-compatible CID addressing)**: Replicates every signed passport version to an independent third-party backup host and peer registries for decentralised resolution.

```
+-----------------------------------------------------------------------------------------+
|                                   DATA PRODUCERS                                        |
|                                                                                         |
|   [Suppliers (VC 2.0)]      [Manufacturer ERP / MES]      [OEM BMS Gateways (MQTT 5)]   |
+-----------------------------------------------------------------------------------------+
          |                              |                                |
          v                              v                                v
+---------------------+   +----------------------------+   +-----------------------------+
| CREDENTIAL VERIFIER |   |   PASSPORT REGISTRY (TS)   |   |  TELEMETRY PIPELINE (Kafka) |
| - DID resolution    |-->| - ID minting (GTIN + SN)   |<--| - SoH plausibility filter   |
| - Status List check |   | - Lifecycle state machine  |   | - TimescaleDB raw readings  |
+---------------------+   | - Succession linkage       |   +-----------------------------+
                          +----------------------------+
                                         | (Kafka: passport.version.published)
                    +--------------------+--------------------+
                    |                                         |
                    v                                         v
+---------------------------------------+   +---------------------------------------------+
|     GS1 DIGITAL LINK RESOLVER (Go)    |   |       FEDERATION & BACKUP REPLICATOR        |
|  - OpenID4VP tier authentication      |   |  - Third-party backup host (RPO <= 5 min)   |
|  - Tier-filtered JSON-LD / HTML       |   |  - Content-addressed signed snapshots       |
+---------------------------------------+   +---------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Passport Registry (`battery_passports`)
```sql
CREATE TABLE battery_passports (
    id UUID PRIMARY KEY,                          -- UUIDv7 internal key
    gtin CHAR(14) NOT NULL CHECK (gtin ~ '^[0-9]{14}$'),
    serial_number VARCHAR(20) NOT NULL,
    digital_link_uri TEXT NOT NULL UNIQUE,
    battery_category VARCHAR(16) NOT NULL CHECK (battery_category IN ('EV', 'LMT', 'INDUSTRIAL')),
    rated_energy_kwh NUMERIC(10, 3) NOT NULL CHECK (rated_energy_kwh > 0),
    chemistry VARCHAR(32) NOT NULL,               -- NMC811, LFP, NCA, LMFP, Na-ion
    economic_operator_id VARCHAR(64) NOT NULL,
    lifecycle_status VARCHAR(16) NOT NULL DEFAULT 'ORIGINAL'
        CHECK (lifecycle_status IN ('ORIGINAL', 'REPURPOSED', 'REMANUFACTURED', 'WASTE', 'RECYCLED')),
    passport_state VARCHAR(16) NOT NULL DEFAULT 'ACTIVE' CHECK (passport_state IN ('ACTIVE', 'SUPERSEDED')),
    predecessor_id UUID REFERENCES battery_passports(id),
    physical_battery_id UUID NOT NULL,            -- stable across succession
    current_version INTEGER NOT NULL DEFAULT 1 CHECK (current_version >= 1),
    snapshot_sha256 CHAR(64) NOT NULL,
    placed_on_market_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    UNIQUE (gtin, serial_number),
    CHECK (battery_category <> 'INDUSTRIAL' OR rated_energy_kwh > 2.000)
);

CREATE UNIQUE INDEX one_active_passport_per_battery
    ON battery_passports (physical_battery_id) WHERE passport_state = 'ACTIVE';
```

### 2.2 Attribute Values & Provenance (`passport_attributes`)
```sql
CREATE TABLE passport_attributes (
    id BIGSERIAL PRIMARY KEY,
    passport_id UUID NOT NULL REFERENCES battery_passports(id),
    attribute_key VARCHAR(96) NOT NULL,           -- e.g. carbon_footprint.total_kgco2e_per_kwh
    access_tier VARCHAR(24) NOT NULL CHECK (access_tier IN ('PUBLIC', 'LEGITIMATE_INTEREST', 'AUTHORITY')),
    value_numeric NUMERIC(18, 6),
    value_text TEXT,
    unit VARCHAR(24),                             -- kgCO2e/kWh, %, cycles, kWh
    is_dynamic BOOLEAN NOT NULL DEFAULT FALSE,
    source_vc_id TEXT,                            -- W3C VC id; NULL only for BMS-derived values
    issuer_did TEXT,
    proof_verified_at TIMESTAMPTZ,
    valid_from TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CHECK (value_numeric IS NOT NULL OR value_text IS NOT NULL),
    CHECK (is_dynamic OR (source_vc_id IS NOT NULL AND proof_verified_at IS NOT NULL)),
    CHECK (attribute_key NOT LIKE 'recycled_content.%' OR value_numeric BETWEEN 0 AND 100)
);
```

## 3. Tier-Filtered Passport Resolution Algorithm
```typescript
type AccessTier = 'PUBLIC' | 'LEGITIMATE_INTEREST' | 'AUTHORITY';

const TIER_GRANTS: Record<AccessTier, ReadonlySet<AccessTier>> = {
  PUBLIC: new Set(['PUBLIC']),
  LEGITIMATE_INTEREST: new Set(['PUBLIC', 'LEGITIMATE_INTEREST']),
  AUTHORITY: new Set(['PUBLIC', 'LEGITIMATE_INTEREST', 'AUTHORITY']),
};

export async function resolvePassport(
  uri: string,
  callerTier: AccessTier,
  repo: PassportRepository,
  catalogue: AttributeCatalogue,
): Promise<PassportView> {
  const match = /\/01\/(\d{14})\/21\/([\x21-\x7E]{1,20})$/.exec(new URL(uri).pathname);
  if (!match) throw new ResolverError(400, 'INVALID_DIGITAL_LINK');
  const [, gtin, serial] = match;
  if (!isValidGtinCheckDigit(gtin)) throw new ResolverError(400, 'INVALID_GTIN_CHECK_DIGIT');

  let passport = await repo.findByGtinSerial(gtin, decodeURIComponent(serial));
  if (!passport) throw new ResolverError(404, 'PASSPORT_NOT_FOUND');

  // Follow succession chain so an old QR label lands on the active passport
  const lineage: string[] = [passport.id];
  while (passport.passportState === 'SUPERSEDED') {
    passport = await repo.findSuccessor(passport.id);
    if (!passport || lineage.includes(passport.id)) throw new ResolverError(500, 'BROKEN_LINEAGE');
    lineage.push(passport.id);
  }

  const allowed = TIER_GRANTS[callerTier];
  const attributes = (await repo.loadAttributes(passport.id, passport.currentVersion)).filter((a) => {
    const def = catalogue.get(a.attributeKey);
    // Default-deny: unknown or untagged attributes are never served
    return def !== undefined && def.tier === a.accessTier && allowed.has(a.accessTier);
  });

  return {
    passportId: passport.id,
    digitalLinkUri: passport.digitalLinkUri,
    lifecycleStatus: passport.lifecycleStatus,
    predecessorIds: lineage.slice(0, -1),
    snapshotSha256: passport.snapshotSha256,
    attributes,
  };
}
```
