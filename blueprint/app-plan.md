# App Plan — Enterprise AI Data Copilot Suite

## App Overview

An agentic AI platform that lets a user talk to their enterprise data in natural language and get back trustworthy answers — not just SQL results, but assessments of whether the underlying data can be trusted, what it means, and whether it's being handled compliantly. Four specialized agents (Data Quality, Catalog, Analytics, Governance) sit behind one chat interface, each backed by a real deterministic engine so the LLM is never asked to invent ground truth.

Built against a synthetic pharma/clinical-trial dataset (subjects, sites, adverse events) so the demo doubles as a regulated-industry (GxP / HIPAA / GDPR / 21 CFR Part 11) data-trust story — a generalized, inspectable version of an FDA-submission-validation platform.

**Who it's for:** the portfolio audience is technical hiring managers/interviewers evaluating whether the builder understands production AI-agent architecture in regulated data environments — not end users of a commercial product. Every design choice should read as "this person has shipped something like this before."

## Tech Stack

- **Langflow 1.10** — visual LangChain canvas; hosts the four module flows (RAG retrieval, NL→SQL generation, classification, narrative generation). Exposes each flow as an MCP tool.
- **n8n 2.0** — orchestration layer: scheduled triggers, source pulls, query routing, human-in-the-loop approval steps, report delivery, per-step audit logging. Native LangChain nodes + supervisor AI Agent node.
- **MCP (Model Context Protocol)** — the bridge between n8n and Langflow; each Langflow flow is consumed as an MCP tool rather than via ad hoc HTTP calls.
- **Postgres + pgvector** — single datastore for relational data, catalog metadata (table/column descriptions, ownership, lineage edges), and vector embeddings.
- **ydata-profiling / Great Expectations** — deterministic data-quality profiling and rule checks.
- **sqlglot** — parses and validates LLM-generated SQL (blocks writes, forces `LIMIT`) before execution against a read-only connection.
- **Microsoft Presidio** (+ regex/NER) — deterministic PII/PHI detection on column samples.
- **Claude API** — the LLM used across all four modules' interpretation layers.
- **Next.js** — chat UI, calls the n8n webhook.
- **Docker Compose** — local orchestration for all services (Postgres, n8n, Langflow); supports a "data never leaves the VPC" story.

## Pages

| Route | Purpose |
|---|---|
| `/` | Landing page — project overview, links to the chat interface and to a short architecture explainer |
| `/chat` | Main copilot interface — single chat box that routes to whichever of the four agents fits the query |
| `/reports` | List of generated reports (data quality, governance/compliance) with timestamps and status |
| `/reports/[id]` | Detail view of a single report — findings, severity, remediation suggestions |
| `/catalog` | Browsable data catalog — tables, columns, ownership, lineage, with a search box backed by the Catalog Agent |

## User Flows

**Chat query flow (core flow — replaces signup/login, this app has no auth in v1):**
1. User types a natural-language question into `/chat` (e.g. "which sites have the most missing adverse-event dates?").
2. Next.js posts the message to the n8n webhook.
3. n8n's supervisor AI Agent node classifies intent and picks the matching MCP tool (Data Quality / Catalog / Analytics / Governance flow in Langflow).
4. The selected Langflow flow runs: deterministic engine produces ground truth → LLM interprets/narrates.
5. n8n logs the step (tool selected, inputs, outputs) to the audit trail and returns the response to the chat UI.
6. Response renders in chat — text, and a chart/table if the Analytics Agent was invoked.

**Scheduled data-quality scan flow:**
1. n8n cron trigger fires on schedule.
2. n8n pulls the target table from Postgres.
3. n8n calls the Data Quality Agent (ydata-profiling/Great Expectations via Langflow flow).
4. Structured violations are turned into a plain-English report by the LLM layer.
5. n8n stores the report and makes it available at `/reports`.

**Scheduled governance/compliance scan flow:**
1. n8n cron trigger fires on schedule.
2. n8n pulls column samples from target tables.
3. n8n calls the Governance Agent — Presidio detects PII/PHI, LLM classifies sensitivity and maps findings to HIPAA/GDPR/21 CFR Part 11.
4. n8n aggregates findings across tables into one compliance report, stored and surfaced at `/reports`.

**Catalog lookup flow:**
1. User searches `/catalog` or asks a lineage/metadata question in `/chat`.
2. Catalog Agent decides: semantic question → pgvector similarity search; lineage/relational question ("what feeds table X?") → structured query against the metadata graph in Postgres.
3. LLM composes the answer from whichever source(s) matched.

**Analytics (NL→SQL) flow:**
1. User asks an analytical question in `/chat`.
2. Analytics Agent flow generates SQL against the known schema.
3. `sqlglot` validates the SQL (read-only, no writes, `LIMIT` enforced); rejected queries are regenerated or surfaced as an error.
4. Validated SQL runs against a read-only Postgres connection.
5. Result set is rendered as a chart; a second LLM pass narrates the insight.

## Database Tables

**Domain data (synthetic clinical-trial dataset):**
- `subjects` — subject_id, site_id, enrollment_date, status, demographic fields (deliberately includes some PII for the Governance Agent to catch)
- `sites` — site_id, name, region, principal_investigator
- `adverse_events` — event_id, subject_id, event_date, severity, description, resolved_date (deliberately seeded with nulls/type mismatches for the Data Quality Agent to catch)

**Metadata / catalog:**
- `catalog_tables` — table_name, description, owner, source_system
- `catalog_columns` — column_id, table_name, column_name, description, data_type, is_pii (flag set by Governance Agent findings)
- `lineage_edges` — source_table, target_table, transform_description

**Vector store:**
- `embeddings` — id, source_type (table/column/doc), source_id, content, embedding (pgvector column)

**Operational / audit:**
- `reports` — report_id, module (quality/governance), generated_at, summary, status, raw_findings (jsonb)
- `audit_log` — log_id, timestamp, agent_invoked, input_query, tool_used, output_summary (populated by n8n per-step logging)

## API Routes

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/chat` | Next.js route that forwards a chat message to the n8n webhook and streams the response back |
| GET | `/api/reports` | List all generated reports (quality + governance) |
| GET | `/api/reports/:id` | Fetch a single report's full detail |
| GET | `/api/catalog` | List/search catalog tables and columns |
| GET | `/api/catalog/:table/lineage` | Fetch lineage edges for a given table |

(n8n owns its own webhook endpoint, e.g. `POST /webhook/copilot-chat`, which the `/api/chat` route calls internally.)

## Components

**Chat (`/chat`):**
- `ChatWindow` — message list + input box
- `MessageBubble` — renders text, or a `ChartResult` / `TableResult` when the Analytics Agent responds
- `AgentBadge` — small indicator showing which of the four agents handled a given response (transparency into routing)

**Reports (`/reports`, `/reports/[id]`):**
- `ReportList` — table of report_id, module, date, status
- `ReportDetail` — findings list with severity badges, remediation suggestions
- `SeverityBadge` — color-coded severity indicator

**Catalog (`/catalog`):**
- `CatalogSearch` — search input backed by the Catalog Agent
- `TableCard` — table name, description, owner, PII flag
- `LineageGraph` — simple visual of upstream/downstream tables for a selected table

**Shared:**
- `Layout` / `NavBar` — links between Chat, Reports, Catalog
- `ArchitectureDiagram` — static component on `/` showing the Langflow/n8n/MCP diagram from the README

## Langflow / n8n Integration

Each module is built as a Langflow flow and exposed as an MCP tool:
- **Data Quality flow:** input = table name → runs profiling engine (as a Langflow custom component wrapping ydata-profiling/Great Expectations) → structured violations → LLM summarization component → output = report text + severity.
- **Catalog flow:** input = NL question → router component decides vector search vs. structured lineage query → retrieval → LLM composition → output = answer.
- **Analytics flow:** input = NL question → schema-aware SQL generation component → sqlglot validation component (custom Python component) → execute against read-only connection → chart-rendering step → LLM narrative component → output = SQL, chart data, narrative.
- **Governance flow:** input = table name → Presidio scan component → LLM classification/compliance-mapping component → output = report text + framework mapping (HIPAA/GDPR/21 CFR Part 11).

n8n consumes these four flows as MCP tools from its supervisor AI Agent node. In Phase 3 (see Build Phases), n8n becomes the sole supervisor; before that, a Langflow Smart Router component can dispatch between flows for faster iteration during Phases 1–2.

n8n also owns:
- Cron triggers for the two scheduled scans (quality, governance)
- The chat webhook that the Next.js frontend calls
- Writing to the `audit_log` table on every agent invocation

## File Parsing

Not applicable in v1 — this suite operates on structured tabular data already loaded into Postgres, not on uploaded documents. (If a future phase adds document-based governance scanning — e.g. scanning PDFs for PII — parsing would be added to the Governance Agent flow at that point, not before.)

## Environment Variables

```
# Postgres
DATABASE_URL=postgres://user:password@localhost:5432/copilot
DATABASE_URL_READONLY=postgres://readonly_user:password@localhost:5432/copilot

# Claude API
ANTHROPIC_API_KEY=

# Langflow
LANGFLOW_API_URL=http://localhost:7860
LANGFLOW_API_KEY=

# n8n
N8N_WEBHOOK_URL=http://localhost:5678/webhook/copilot-chat
N8N_API_KEY=

# Next.js
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

## Build Phases

**Phase 0 — Foundation (~1 week)**
- Generate the synthetic pharma/clinical dataset (subjects, sites, adverse_events), seeded with quality issues (nulls, duplicates, type mismatches) and PII/PHI fields.
- Stand up Postgres + pgvector, Langflow, and n8n in Docker Compose.
- Create the `catalog_tables`, `catalog_columns`, `lineage_edges`, `embeddings`, `reports`, and `audit_log` tables.

**Phase 1 — One vertical slice: Analytics Agent (~1 week)**
- Build the full path end to end: chat UI → n8n webhook → Langflow NL→SQL flow → sqlglot validation → read-only execution → chart → LLM narrative.
- This proves the whole architectural pattern before replicating it three more times.

**Phase 2 — Remaining three agents (~2 weeks)**
- Catalog Agent (hybrid vector + structured lineage RAG), then Data Quality Agent, then Governance Agent.
- Each reuses the n8n/Langflow/MCP plumbing from Phase 1.

**Phase 3 — The suite (~1 week)**
- Add the n8n supervisor AI Agent node as the single router across all four MCP tools, replacing the Phase 1–2 Langflow Smart Router.
- Wire up per-step audit logging to `audit_log`.
- Build out `/reports` and `/catalog` pages in the Next.js frontend.

**Phase 4 — Portfolio packaging (~1 week)**
- Architecture writeup (reuse/expand the README diagram).
- 3–4 minute demo video (Loom) walking through a chat session that hits all four agents.
- Metrics/results summary line for the portfolio and resume.
- Build-in-public content series.
