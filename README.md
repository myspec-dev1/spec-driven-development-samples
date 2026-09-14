# spec-driven-development-samples

A collection of sample Spec-Driven Development (SDD) specifications generated with [MySpec.dev](https://myspec.dev).

Each sample under `specs/` is a complete specification set for a software project, written before any code exists. Every sample contains the same four documents:

| File | Purpose |
| --- | --- |
| `constitution.md` | Project vision, non-negotiable principles, and technology constraints |
| `requirements.md` | What the system must do: functional and non-functional requirements |
| `solution.md` | How it will be built: architecture, design decisions, and module layout |
| `tasks.md` | An ordered, milestone-grouped implementation task list with acceptance criteria |

## Samples

- [**binance-trading-bot-optimiser**](specs/binance-trading-bot-optimiser/) — A local, non-interactive Python CLI that ingests Binance public historical market data into DuckDB and backtests or optimises Spot Grid and Futures Grid bot configurations using walk-forward validation. The design centres on discounting its own optimism: it reports fee-accurate results, applies deflation for search size, and never executes live orders or touches an account.

- [**clinical-trial-ctms-etmf**](specs/clinical-trial-ctms-etmf/) — A compliant Clinical Trial Management System (CTMS) and Electronic Trial Master File (eTMF) for decentralized and site-based clinical research. Built to FDA 21 CFR Part 11 and ICH GCP standards, it enforces cryptographic dual-factor digital signatures, Merkle-hashed computer-generated audit trails, complete DIA TMF Reference Model v3.2 taxonomy (Zones 01–11), automated protocol deviation tracking, and a 24-hour Serious Adverse Event (SAE) escalation engine with CIOMS I / MedWatch 3500A exports.

- [**cloud-erp-mes**](specs/cloud-erp-mes/) — A modular cloud ERP and shop-floor Manufacturing Execution System (MES) designed for precision discrete manufacturing. Enforces a strict double-entry general ledger invariant ($\sum \text{Debits} = \sum \text{Credits}$), multi-level recursive Bill of Materials (BOM) cost rollups, and Engineering Change Order (ECO) locks, coupled with an offline-tolerant edge gateway running MQTT/OPC-UA machine telemetry and real-time Overall Equipment Effectiveness (OEE) calculation.

- [**csrd-esrs-sustainability**](specs/csrd-esrs-sustainability/) — A statutory sustainability reporting and corporate governance platform built for mandatory EU Corporate Sustainability Reporting Directive (CSRD) and CSDDD compliance. Automates two-dimensional quantitative Double Materiality Assessments (DMA) across all 10 topical ESRS standards (E1–E5, S1–S4, G1), Scope 1–3 GHG Protocol carbon accounting with versioned emission factor databases, cryptographic raw evidence lineage DAGs, and ESEF-compliant Inline XBRL (iXBRL) digital taxonomy filing exports for statutory external assurance (ISAE 3000 / ISSA 5000).

- [**enterprise-ai-gateway**](specs/enterprise-ai-gateway/) — A high-performance, zero-trust streaming reverse proxy and policy enforcement platform for enterprise LLM traffic and autonomous agent tools. Built for EU AI Act, ISO 42001, and SOC 2 compliance, it adds $\le 15\text{ms}$ latency overhead while executing inline PII/PHI redaction, prompt injection defense, dynamic provider failover, semantic caching, token-level FinOps budgets, and an MCP agent execution sandbox with Human-in-the-Loop authorization and tamper-evident WORM audit logs.

- [**go-htmx-template-engine**](specs/go-htmx-template-engine/) — `.ghtmx`, an open source Go template engine hard-forked from `templ` that makes htmx a first-class, compile-checked language concept. Bindings such as `hx-get` reference Go handler symbols resolved against a statically derived route table, page fragments get dual render entry points at build time, and `HX-Trigger` events are declared with typed payloads. Ships a CLI, runtime, LSP, dev server, and adapters for common Go routers.

- [**payment-reconciliation-engine**](specs/payment-reconciliation-engine/) — A high-throughput financial reconciliation and dispute defense engine processing tens of millions of records across heterogeneous payment rails (card schemes, ACH, FedNow, SEPA Instant, and ISO 20022 camt.053 / MT940). Features tiered deterministic 1:1, 1:N, and N:M matching algorithms, automated zero-float suspense break aging, balanced ERP journal postings, and automated chargeback evidence assembly with statutory representment deadline tracking.

- [**warehouse-management-wms**](specs/warehouse-management-wms/) — A high-velocity Warehouse Management System (WMS) and multimodal freight logistics platform. Combines 3D coordinate bin addresses, GS1-128 License Plate Numbering (LPN), and traveling salesperson pick-path optimization with an intermodal freight state machine across ocean, air, rail, and drayage legs, complete with an automated demurrage/detention countdown watchdog and bi-directional ANSI X12 (204, 214, 304) and UN/EDIFACT (IFTMIN, IFTSTA, DESADV) bridges.
