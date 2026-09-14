# Solution Architecture: Clinical Trial Management System (CTMS) & 21 CFR Part 11 Compliant eTMF

## 1. System Architecture Overview

The solution implements a cloud-validated microservices architecture deployed within a HIPAA-compliant AWS / Azure enclave:
1. **Core CTMS & Subject Engine**: NestJS / TypeScript backend with PostgreSQL 16 enforcing strict multi-tenant study row-level security (RLS).
2. **eTMF Compliant Storage & PDF Engine**: S3 Object Lock (Compliance Mode) with LibreOffice / PDF-lib worker cluster for cryptographic watermarking and 21 CFR Part 11 signature stamping.
3. **Regulatory Audit Trail Service**: Append-only event store backed by AWS QLDB or immutable PostgreSQL hypertable with SHA-256 Merkle tree hashing.
4. **Safety Escalation Engine**: Temporal.io / BullMQ durable workflows ensuring 24-hour SAE notification countdown timers survive system restarts.

```
+-----------------------------------------------------------------------------------------+
|                                    APPLICATION CLIENTS                                  |
|                                                                                         |
|   [CRA Monitoring Portal]      [Investigator Tablet]       [Auditor Inspection Room]    |
+-----------------------------------------------------------------------------------------+
                                          | (TLS 1.3 / FIDO2 WebAuthn)
                                          v
+-----------------------------------------------------------------------------------------+
|                                     API GATEWAY                                         |
|                 (JWT Session Validator, 15-Min Inactivity Timeout, RBAC)                |
+-----------------------------------------------------------------------------------------+
       |                                  |                                   |
       v                                  v                                   v
+------------------+             +------------------+               +------------------+
|  CTMS & Subjects |             |   eTMF Engine    |               |  Pharmacovigilance|
|  - Visit Windows |             |  - DIA v3.2 Tree |               |  - 24h SAE Timers|
|  - Site Greenlight|            |  - Part 11 Sign  |               |  - CIOMS Exports |
+------------------+             +------------------+               +------------------+
       |                                  |                                   |
       +----------------------------------+-----------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                              DATA & COMPLIANCE PERSISTENCE                              |
|                                                                                         |
|  [PostgreSQL 16: Multi-Tenant RLS] <---> [S3 WORM: 21 CFR Part 11 Documents & Manifest] |
|                                                                                         |
|  [Audit Event Ledger (Merkle Hashed)] <---> [Temporal.io: Durable Escalation Workflows] |
+-----------------------------------------------------------------------------------------+
```

## 2. 21 CFR Part 11 Cryptographic Signature Specification

### 2.1 Dual-Factor Signature Flow
1. User clicks **"Sign Document"** in the eTMF viewer.
2. Server challenges user for credential confirmation:
   - Password hash verification OR WebAuthn biometric assertion.
   - Purpose selection from standardized enumerated intents: `['AUTHORED', 'REVIEWED', 'APPROVED', 'VERIFIED']`.
3. Server generates canonical document hash:
   $$H_{\text{doc}} = \text{SHA256}(\text{DocumentBytes})$$
4. Server generates signature record:
   ```json
   {
     "document_id": "doc_etmf_9921",
     "document_version": 2,
     "document_sha256": "8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4",
     "signer_user_id": "usr_cra_dr_miller",
     "signer_legal_name": "Dr. Sarah Miller, MD",
     "signer_role": "Principal Investigator",
     "timestamp_utc": "2026-09-14T12:00:00.000Z",
     "signature_intent": "APPROVED",
     "ip_address": "192.0.2.45"
   }
   ```
5. Server signs payload using private asymmetric key stored in AWS KMS / Azure Key Vault (HSM), generating signature hash $S_{\text{sig}}$.
6. PDF worker appends an immutable visual signature certification page and locks PDF editing permissions (ISO 32000-1 certified signature).

## 3. DIA TMF Reference Model Database Schema

```sql
CREATE TABLE etmf_zones (
    zone_code VARCHAR(2) PRIMARY KEY,
    zone_name VARCHAR(128) NOT NULL
);

CREATE TABLE etmf_sections (
    section_code VARCHAR(8) PRIMARY KEY,
    zone_code VARCHAR(2) NOT NULL REFERENCES etmf_zones(zone_code),
    section_name VARCHAR(128) NOT NULL
);

CREATE TABLE etmf_artifacts (
    artifact_code VARCHAR(16) PRIMARY KEY,
    section_code VARCHAR(8) NOT NULL REFERENCES etmf_sections(section_code),
    artifact_name VARCHAR(256) NOT NULL,
    is_mandatory_by_default BOOLEAN DEFAULT true NOT NULL
);

CREATE TABLE etmf_documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    study_id UUID NOT NULL,
    site_id UUID, -- Nullable for global trial-level documents
    artifact_code VARCHAR(16) NOT NULL REFERENCES etmf_artifacts(artifact_code),
    file_name VARCHAR(255) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    sha256_hash VARCHAR(64) NOT NULL,
    s3_object_key VARCHAR(512) NOT NULL,
    status VARCHAR(32) NOT NULL CHECK (status IN ('DRAFT', 'UPLOADED', 'QC_PENDING', 'QC_PASSED', 'APPROVED', 'SUPERSEDED')),
    created_by UUID NOT NULL,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

-- Immutable Computer-Generated Audit Trail
CREATE TABLE regulatory_audit_log (
    id BIGSERIAL PRIMARY KEY,
    study_id UUID NOT NULL,
    entity_type VARCHAR(64) NOT NULL,
    entity_id VARCHAR(64) NOT NULL,
    action VARCHAR(32) NOT NULL, -- CREATE, READ, UPDATE, SIGN, DOWNLOAD, REJECT
    user_id UUID NOT NULL,
    user_role VARCHAR(64) NOT NULL,
    previous_state JSONB,
    new_state JSONB,
    reason_for_change TEXT,
    event_sha256 VARCHAR(64) NOT NULL, -- Merkle chain: SHA256(previous_event_sha256 + payload)
    timestamp_utc TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);
```

## 4. 24-Hour SAE Escalation Workflow Engine
- Implemented using Temporal.io durable state machines:
  - **Activity 1**: Receive Adverse Event payload; verify Seriousness criteria = `true`.
  - **Activity 2**: Dispatch instant priority push notification and email to Lead Medical Monitor.
  - **Timer 1**: Sleep 18 hours awaiting `ACKNOWLEDGE_SAFETY_EVENT` signal.
  - **Activity 3 (Escalation)**: If unacknowledged after 18 hours, initiate automated voice call (Twilio) and SMS alert to Backup Safety Officer.
  - **Timer 2**: Sleep 6 hours (total 24 hours).
  - **Activity 4**: If still unacknowledged at 24 hours, generate automatic Critical Audit Incident and flag study dashboard for urgent intervention.
