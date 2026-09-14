# Solution Architecture: Financial Reconciliation & Multi-Rail Dispute Engine

## 1. System Architecture Overview

The system is architected for ultra-high throughput, mathematical precision, and zero-downtime reliability:
1. **Streaming Ingestion & Normalizer (Rust / Go)**: Ingests raw settlement files, validates checksums, parses banking and card schemas, and emits normalized records into Kafka.
2. **Deterministic Match Engine (Apache Flink / ClickHouse / PostgreSQL)**: Performs distributed join operations across sliding time windows, executing tiered matching algorithms with zero cross-tenant lock contention.
3. **Dispute Automation Service (Node.js / Python LLM Worker)**: Extracts transaction evidence from merchant data stores, queries delivery APIs, and generates network-compliant dispute defense PDFs.
4. **Suspense & GL Integration API (NestJS / PostgreSQL)**: Maintains suspense account balances, aging schedules, and exports balanced journal entries to enterprise ERPs.

```
+-----------------------------------------------------------------------------------------+
|                                    DATA INGESTION FEEDS                                 |
|                                                                                         |
|   [Banking SFTP (MT940/CAMT)]    [Card Schemes (Visa/MC)]    [PSP APIs (Stripe/Adyen)]  |
+-----------------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                          STREAMING INGESTION & DE-DUPE (Go)                             |
|                                                                                         |
|  [File Decryption (PGP)] ---> [Schema Parser] ---> [Fingerprint De-dupe (Redis)]        |
+-----------------------------------------------------------------------------------------+
                                          | (Kafka: payment.settlement.normalized)
                                          v
+-----------------------------------------------------------------------------------------+
|                              DETERMINISTIC MATCH ENGINE                                 |
|                                                                                         |
|  [Tier 1: Exact 1:1 Engine] ---> [Tier 2: 1:N Batch Joiner] ---> [Tier 3: FX Tolerances]|
+-----------------------------------------------------------------------------------------+
                                          |
                     +--------------------+--------------------+
                     |                                         |
                     v                                         v
+---------------------------------------+   +---------------------------------------------+
|          RECONCILED LEDGER            |   |               EXCEPTION & SUSPENSE          |
|  - Immutable Match Journal            |   |  - Aging Dashboard (0-7d, 8-30d, 30+d)      |
|  - ERP GL Balanced Journals           |   |  - Dispute & Chargeback Auto-Defense Engine |
+---------------------------------------+   +---------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Normalized Transaction Ledger (`recon_records`)
```sql
CREATE TABLE recon_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type VARCHAR(32) NOT NULL CHECK (source_type IN ('INTERNAL_ORDER', 'BANK_STATEMENT', 'PROCESSOR_SETTLEMENT')),
    source_channel VARCHAR(32) NOT NULL, -- FEDNOW, SEPA, VISA, STRIPE, SWIFT
    external_reference_id VARCHAR(128) NOT NULL,
    account_id VARCHAR(64) NOT NULL,
    transaction_time TIMESTAMPTZ NOT NULL,
    settlement_date DATE NOT NULL,
    currency VARCHAR(3) NOT NULL,
    gross_amount NUMERIC(18, 4) NOT NULL,
    fee_amount NUMERIC(18, 4) DEFAULT 0.0000 NOT NULL,
    net_amount NUMERIC(18, 4) NOT NULL,
    match_status VARCHAR(32) NOT NULL DEFAULT 'UNMATCHED' CHECK (match_status IN ('UNMATCHED', 'MATCHED', 'SUSPENSE_BREAK', 'DISPUTED')),
    record_hash VARCHAR(64) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    UNIQUE (source_type, source_channel, external_reference_id, settlement_date, gross_amount)
);
```

### 2.2 Match Journal & Audit Ledger (`recon_matches`)
```sql
CREATE TABLE recon_matches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    match_rule_tier VARCHAR(16) NOT NULL CHECK (match_rule_tier IN ('TIER_1_EXACT', 'TIER_2_GROUP', 'TIER_3_TOLERANCE', 'MANUAL_OVERRIDE')),
    internal_record_id UUID NOT NULL REFERENCES recon_records(id),
    external_record_id UUID NOT NULL REFERENCES recon_records(id),
    variance_amount NUMERIC(18, 4) DEFAULT 0.0000 NOT NULL,
    confidence_score NUMERIC(5, 4) NOT NULL,
    matched_by VARCHAR(64) NOT NULL, -- SYSTEM_ENGINE or user UUID
    matched_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);
```

## 3. Tier 1 & Tier 2 Matching Algorithm
```typescript
export async function executeTier1ExactMatch(
  internalRecords: ReconRecord[],
  settlementRecords: ReconRecord[]
): Promise<{ matched: MatchedPair[]; remainingInternal: ReconRecord[]; remainingExternal: ReconRecord[] }> {
  const externalMap = new Map<string, ReconRecord>();
  
  // Index settlement records by Composite Key: externalReferenceId + normalizedAmount
  for (const ext of settlementRecords) {
    const key = `${ext.externalReferenceId}|${ext.netAmount.toFixed(4)}|${ext.currency}`;
    externalMap.set(key, ext);
  }

  const matched: MatchedPair[] = [];
  const remainingInternal: ReconRecord[] = [];

  for (const internal of internalRecords) {
    const key = `${internal.externalReferenceId}|${internal.netAmount.toFixed(4)}|${internal.currency}`;
    if (externalMap.has(key)) {
      const ext = externalMap.get(key)!;
      matched.push({
        matchRuleTier: 'TIER_1_EXACT',
        internalRecordId: internal.id,
        externalRecordId: ext.id,
        varianceAmount: 0,
        confidenceScore: 1.0,
      });
      externalMap.delete(key);
    } else {
      remainingInternal.push(internal);
    }
  }

  return {
    matched,
    remainingInternal,
    remainingExternal: Array.from(externalMap.values()),
  };
}
```
