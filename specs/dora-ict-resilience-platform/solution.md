# Solution Architecture: DORA ICT Risk & Operational Resilience Platform

## 1. System Architecture Overview

The system is architected for regulatory determinism, evidence integrity, and deadline reliability:
1. **Incident Intake & Classification Service (TypeScript / NestJS)**: Receives ITSM/SIEM webhooks, enriches incidents with affected CIFs from the dependency graph, and evaluates a versioned RTS ruleset as a pure function.
2. **Deadline Watchdog (Go / Temporal)**: Runs durable timers per major incident for initial, intermediate, and final reports with escalation at 50/75/90% of each window via PagerDuty, Teams, and email.
3. **ICT Inventory & Dependency Graph (PostgreSQL + Apache AGE)**: Stores assets, functions, ICT services, providers, and subcontracting chains; serves transitive dependency and concentration (HHI) queries.
4. **Register of Information & xBRL-CSV Builder (Python / Arelle)**: Maps register tables to ITS templates, validates LEIs against the GLEIF API, runs ESA validation rules, and packages xBRL-CSV report packages.
5. **Resilience Testing & Exit Plan Tracker (NestJS / PostgreSQL)**: Manages the annual testing programme, TLPT (TIBER-EU) phases, findings remediation, and exit-plan tests.
6. **Evidence Vault (S3 Object Lock, compliance mode)**: Hash-chained, WORM storage for submitted reports, register snapshots, and TLPT attestations.

```
+-----------------------------------------------------------------------------------------+
|                                     SOURCE SYSTEMS                                      |
|                                                                                         |
|   [ServiceNow / Jira SM]     [Sentinel / Splunk SIEM]     [CMDB / Contract Mgmt (CLM)]  |
+-----------------------------------------------------------------------------------------+
                  |                         |                            |
                  v                         v                            v
+---------------------------------------------------+   +---------------------------------+
|        INCIDENT INTAKE & CLASSIFICATION (TS)      |   |  ICT INVENTORY & DEPENDENCY     |
|                                                   |   |  GRAPH (PostgreSQL + AGE)       |
|  [Webhook API] -> [CIF Enricher] -> [RTS Engine]  |<->|  Function -> Asset -> Service   |
+---------------------------------------------------+   |  -> Provider -> Subcontractor   |
                  | (Kafka: dora.incident.classified)   +---------------------------------+
                  v                                                      |
+---------------------------------------------------+   +---------------------------------+
|           DEADLINE WATCHDOG (Go / Temporal)       |   |  REGISTER OF INFORMATION        |
|                                                   |   |  & xBRL-CSV BUILDER (Arelle)    |
|  [Initial 4h/24h] -> [Interim 72h] -> [Final 1M]  |   |  [LEI/GLEIF] -> [ESA Rules]     |
+---------------------------------------------------+   +---------------------------------+
                  |                                                      |
                  v                                                      v
+-----------------------------------------------------------------------------------------+
|        COMPETENT AUTHORITY GATEWAY  +  EVIDENCE VAULT (S3 Object Lock, SHA-256 chain)   |
+-----------------------------------------------------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 ICT Incidents & Classification (`ict_incidents`)
```sql
CREATE TABLE ict_incidents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_lei CHAR(20) NOT NULL CHECK (entity_lei ~ '^[A-Z0-9]{18}[0-9]{2}$'),
    external_ticket_ref VARCHAR(128) NOT NULL,
    detected_at TIMESTAMPTZ NOT NULL,
    aware_at TIMESTAMPTZ NOT NULL,
    classified_at TIMESTAMPTZ,
    classification VARCHAR(16) NOT NULL DEFAULT 'PENDING' CHECK (classification IN ('PENDING', 'NON_MAJOR', 'MAJOR', 'RECURRING_MAJOR')),
    ruleset_version VARCHAR(32),
    critical_services_affected BOOLEAN NOT NULL DEFAULT FALSE,
    clients_affected_pct NUMERIC(7, 4) CHECK (clients_affected_pct BETWEEN 0 AND 100),
    clients_affected_count INTEGER CHECK (clients_affected_count >= 0),
    transactions_value_eur NUMERIC(20, 2) CHECK (transactions_value_eur >= 0),
    economic_impact_eur NUMERIC(20, 2) CHECK (economic_impact_eur >= 0),
    downtime_minutes INTEGER CHECK (downtime_minutes >= 0),
    member_states_affected CHAR(2)[] NOT NULL DEFAULT '{}',
    root_cause_code VARCHAR(32),
    criteria_evaluation JSONB, -- per-criterion value, threshold, met flag
    prev_hash CHAR(64),
    record_hash CHAR(64) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CHECK (aware_at >= detected_at),
    CHECK (classified_at IS NULL OR classified_at >= aware_at),
    UNIQUE (entity_lei, external_ticket_ref)
);
```

### 2.2 Register of Information: Contractual Arrangements (`roi_contractual_arrangements`)
```sql
CREATE TABLE roi_contractual_arrangements (
    contract_ref VARCHAR(64) PRIMARY KEY,
    entity_lei CHAR(20) NOT NULL,
    provider_id VARCHAR(64) NOT NULL,
    provider_id_type VARCHAR(8) NOT NULL CHECK (provider_id_type IN ('LEI', 'EUID')),
    ict_service_type VARCHAR(16) NOT NULL, -- ITS taxonomy code, e.g. S01..S19
    supports_cif BOOLEAN NOT NULL DEFAULT FALSE,
    substitutability VARCHAR(24) NOT NULL CHECK (substitutability IN ('NOT_SUBSTITUTABLE', 'HIGHLY_COMPLEX', 'MEDIUM_COMPLEXITY', 'EASILY_SUBSTITUTABLE')),
    annual_cost_eur NUMERIC(18, 2) NOT NULL CHECK (annual_cost_eur >= 0),
    governing_law_country CHAR(2) NOT NULL,
    data_storage_country CHAR(2),
    start_date DATE NOT NULL,
    end_date DATE,
    exit_plan_id UUID,
    rank_in_chain SMALLINT NOT NULL DEFAULT 1 CHECK (rank_in_chain >= 1), -- 1 = direct provider
    updated_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CHECK (end_date IS NULL OR end_date >= start_date),
    CHECK (NOT supports_cif OR exit_plan_id IS NOT NULL)
);
```

## 3. RTS Major Incident Classification & Deadline Computation
```typescript
const THRESHOLDS = {
  clientsPct: 10, clientsCount: 100_000, txnPct: 10, txnValueEur: 15_000_000,
  durationHours: 24, criticalDowntimeHours: 2, memberStates: 2, economicEur: 100_000,
} as const;

export function classifyIncident(i: IncidentFacts, rulesetVersion: string): Classification {
  const criteria: CriterionResult[] = [
    { id: 'CLIENTS', met: i.clientsAffectedPct > THRESHOLDS.clientsPct || i.clientsAffectedCount > THRESHOLDS.clientsCount },
    { id: 'TRANSACTIONS', met: i.txnAffectedPct > THRESHOLDS.txnPct || i.txnValueEur > THRESHOLDS.txnValueEur },
    { id: 'DURATION', met: i.durationHours > THRESHOLDS.durationHours || i.criticalDowntimeHours > THRESHOLDS.criticalDowntimeHours },
    { id: 'GEOGRAPHY', met: new Set(i.memberStatesAffected).size >= THRESHOLDS.memberStates },
    { id: 'DATA_LOSS', met: i.materialDataLoss },
    { id: 'ECONOMIC', met: i.economicImpactEur >= THRESHOLDS.economicEur },
    { id: 'REPUTATIONAL', met: i.reputationalImpact },
  ];

  const otherCriteriaMet = criteria.filter((c) => c.met).length;
  const isMajor =
    i.criticalServicesAffected && (i.maliciousUnauthorisedAccess || otherCriteriaMet >= 2);

  return { verdict: isMajor ? 'MAJOR' : 'NON_MAJOR', rulesetVersion, criteria };
}

export function computeReportingDeadlines(awareAt: Date, classifiedAt: Date): ReportingDeadlines {
  const HOUR = 3_600_000;
  const initialDue = new Date(
    Math.min(classifiedAt.getTime() + 4 * HOUR, awareAt.getTime() + 24 * HOUR),
  );
  return {
    initialDue,
    // Intermediate (initial_sent + 72h) and final (last intermediate + 1 month) are
    // scheduled by the Temporal workflow once the preceding report is actually submitted.
    intermediateOffsetMs: 72 * HOUR,
    finalOffsetMonths: 1,
  };
}
```
