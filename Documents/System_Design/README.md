# Enterprise AI Data Copilot Suite

An agentic platform that lets any organization **understand, govern, and analyze** its data in natural language.

Four coordinated agents over a shared semantic layer. Each pairs a **deterministic engine** (ground truth) with an **LLM layer** (explanation). Industry specificity is injected as configuration — not code.

| Agent | Question it answers | Engine |
|---|---|---|
| **Data Quality** | Can I trust this dataset? | ydata-profiling · Soda |
| **Catalog** | What do we have, who owns it, where's it from? | lineage graph + pgvector |
| **Analytics** | What does the data say? | sqlglot AST validator |
| **Governance** | What's sensitive, and are we compliant? | Presidio · regex · NER |

## Why it's different

Most "AI data copilots" point an LLM at a database. That fails on exactly the tasks that matter: an LLM can't reliably *count* nulls, can't *guarantee* a query is read-only, and can't give a defensible precision figure for sensitive-data detection.

Here, engines establish ground truth and the LLM only explains, classifies, and narrates.

## Works for any industry

Compliance frameworks are just rule sets over classified data. Once policy is **data** rather than code, adding an industry is a config PR:

`core` (GDPR/PII) · `financial-services` (SOX, PCI-DSS, GLBA) · `life-sciences` (21 CFR Part 11, HIPAA) · `retail` (CCPA) · `public-sector` (FISMA) · `manufacturing` · `telecom`

Swap the pack and the identical Governance Agent produces a PCI report instead of a Part 11 report.

## Architecture

![System architecture](docs/assets/system-architecture.png)

**→ [Full system design](docs/SYSTEM_DESIGN.md)** — context & container diagrams, domain pack spec, per-agent designs, five annotated data flows, ER model, security model, evaluation harness, ADRs, and build phases.

## Stack

n8n (orchestration + audit) · Langflow (agent reasoning) · MCP (contract boundary) · Postgres + pgvector (control plane) · Presidio · sqlglot · Next.js

## Status

v0.1 — design complete, build targeted for Q3 2026. Contributions and architecture critique welcome.

---
Sankar Kumar Palaniappan · [sankar.work](https://sankar.work)
