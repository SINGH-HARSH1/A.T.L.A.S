# A.T.L.A.S
Automated Task, Logistics, &amp; Assistant System
==========================================================================================================================================
# A.T.L.A.S. — Enterprise Technical Architecture & Google ADK Implementation Guide

> **Automated Task, Logistics, & Assistant System**[cite: 3]  
> *Permission-aware enterprise personal assistant powered by Google Agent Development Kit (ADK) and FastAPI control plane.*

---

## 1. Executive Summary & Positioning

A.T.L.A.S. is a permission-aware enterprise personal assistant designed to turn unstructured user requests into coordinated, auditable, and stateful workflows across personal and professional domains.

Rather than relying on unrestricted single-prompt LLM loops or brittle agent swarms, A.T.L.A.S. enforces a clean separation of concerns: **Agents propose decisions, Workflows constrain execution state, and Tool Gateways execute deterministic side-effects**.

### Core Defensible Capabilities
* **Deterministic Authorization:** Security, RBAC, and approval enforcement are managed in application code, never delegated to model system prompts.
* **Bounded Autonomy:** Strict step limits, token budgets, timeout controls, and explicit human-in-the-loop (HITL) gates prevent run-away execution loops.
* **Durable & Resumable Execution:** State checkpoints persist to PostgreSQL, enabling multi-step workflows to pause for approval or recover gracefully from infrastructure restarts.
* **Source-Grounded Operations:** Knowledge retrieval isolates authoritative data from vector search indices and treats untrusted external text safely.

---

## 2. Taxonomy & Core Design Principles

To prevent prompt bloat and state fragmentation, A.T.L.A.S. establishes a strict four-layer architecture:

| Architectural Layer | Core Responsibility | Google ADK Mapping | Example |
| :--- | :--- | :--- | :--- |
| **Agent** | Reasoning, intent interpretation, adaptive planning, and result synthesis. | `google.adk.Agent` | Evaluating availability across multiple calendars. |
| **Workflow** | Runtime state management, step sequencing, and cycle control. | `google.adk.Workflow` | Executing an approval gate before sending an external email. |
| **Tool** | Deterministic operations with strictly typed inputs, outputs, and permissions. | `@tool` / MCP Adapter | Fetching calendar free/busy slots via Google API. |
| **State / Artifact** | Durable session records, approval tokens, citations, and execution context. | `TaskState` / Session KV | Serialized JSON containing step history and payload hashes. |

### Core Architectural Rules
1. **Interpretation != Execution:** Models propose actions; deterministic tool gateways run them.
2. **Never Trust Untrusted Text:** External data (emails, scraped web pages, file attachments) must be sanitized before passing into agent reasoning loops.
3. **Idempotent Retries:** Network timeouts on external API calls trigger reconciliation checks before re-issuing side-effect actions.

---

## 3. High-Level Architecture & System Blueprint

A.T.L.A.S. combines **Google ADK** (for agent hierarchy, prompt lifecycle, and native confirmation gates) with a **FastAPI Control Plane** (for API routing, security, and persistence).

```text
                       USER & CLIENT INTERFACES
               Web UI / CLI / Webhook / Slack / Mobile
                                  │
                      API & IDENTITY GATEWAY
              OAuth2/OIDC, RBAC, Rate Limiting, Tenant Context
                                  │
                  A.T.L.A.S. FASTAPI CONTROL PLANE
        ┌────────────────────────────────────────────────────┐
        │ Request Router & Intent Parsing Gateway           │
        │ Durable State Lifecycle Manager (PostgreSQL)      │
        │ Policy Engine & Human Approval Gatekeeper          │
        │ Background Worker Queue Interface (Celery/Arq)    │
        └─────────────────────────┬──────────────────────────┘
                                  │
                     GOOGLE ADK WORKFLOW RUNTIME
        ┌─────────────────────────┴──────────────────────────┐
        │  Executive Orchestrator Agent (Root Router)        │
        │                                                    │
        │   ┌─────────────────┐    ┌─────────────────────┐   │
        │   │ Communications  │    │     Calendar &      │   │
        │   │     Agent       │    │  Scheduling Agent   │   │
        │   └────────┬────────┘    └──────────┬──────────┘   │
        │            │                        │              │
        │   ┌────────┴────────┐    ┌──────────┴──────────┐   │
        │   │   Knowledge &   │    │ Tasks & Operations  │   │
        │   │   Media Agent   │    │        Agent        │   │
        │   └─────────────────┘    └─────────────────────┘   │
        └─────────────────────────┬──────────────────────────┘
                                  │
                       TOOL EXECUTION GATEWAY
          Schema Validation | Credential Isolation | Action Audit
                                  │
                    MCP SERVERS & DIRECT ADAPTERS
         Google Workspace | Email Webhooks | Expense DB | Media Worker
                                  │
                              DATA PLANE
          PostgreSQL (State/Audit) | pgvector | S3 Object Storage
```

## 4. Agent & Subagent Ecosystem (Phase 1 Target: Executive Desk)

The initial commercial milestone focuses on A.T.L.A.S. Executive Desk, targeting productivity, daily briefings, and communication drafting.  
1. **Executive Orchestrator (Root Agent):** Interprets complex multi-intent prompts, creates execution plans, delegates sub-tasks to specialized agents, and aggregates cited briefings.  
2. **Communications Agent:** Processes inbox webhooks, triages priority messages, extracts actionable items, and prepares draft replies (never dispatches directly).  
3. **Calendar & Scheduling Agent:** Inspects availability, detects overlaps, and identifies optimal slots while respecting protected focus blocks.  
4. **Knowledge & Media Agent:** Performs semantic search across ingestion stores (pgvector), retrieves source context, and coordinates transcript processing

## 5. State Management & Task Lifecycle
### Task Execution Lifecycle

```text
RECEIVED ──► INTERPRETING ──► PLANNING ──► VALIDATING ──► EXECUTING ──► COMPLETED
                 │                             │              │
                 ▼                             │              ▼
        NEEDS_CLARIFICATION                    │     WAITING_FOR_APPROVAL
                                               │              │
                                               ▼              ▼
                                           REPLANNING     CANCELLED / EXPIRED

```

## 6. Authorization, Action Classification & HITL Approval

A.T.L.A.S. categorizes every tool call into risk-based Action Classes to maintain deterministic boundary enforcement.

Action Class,Risk Level,Examples,Policy Handling
Read-Only,Low,"Inspect calendar, query document RAG index[cite: 2]",Executes automatically within user RBAC scope[cite: 2].
Draft-Only,Low-Med,"Draft email reply, prepare meeting invite payload[cite: 2]",Generates draft artifact; no external dispatch[cite: 2].
Internal Write,Medium,"Log expense record, update internal task status[cite: 2]",Executes under strictly configured policies[cite: 2].
External Commitment,High,"Send email, book travel, submit payment[cite: 2]",Requires explicit signed Human-in-the-Loop approval token[cite: 2].


## 7. Step-by-Step Implementation Roadmap (Phases 0 – 11)

0. **Phase 0** — Target Scope Definition: Establish threat models, user personas, baseline safety policies, and evaluation sets[cite: 2].

1. **Phase 1** — Schema & Contracts: Define TaskState schemas, Pydantic data models, tool input/output specifications, and lifecycle states[cite: 2].

2. **Phase 2** — Platform Skeleton: Build the core FastAPI application with Google ADK agent wrappers, OAuth context, and PostgreSQL state storage[cite: 2].

3. **Phase 3** — Tool Execution Gateway: Register basic tools using @tool annotations and implement authorization and validation wrappers[cite: 2].

4. **Phase 4** — HITL Approval Engine: Connect ADK confirmation_required gates to FastAPI pause/resume routes and signed token verifiers[cite: 2].

5. **Phase 5** — Executive Desk Integration: Deliver the first end-to-end multi-agent execution pipeline (Calendar, Email Triage, Briefings)[cite: 2].

6. **Phase 6** — Async Ingestion Pipeline: Deploy background queue workers (Celery/Arq) for media processing (yt-dlp, Whisper, OCR)[cite: 2].

7. **Phase 7** — Governed Memory Infrastructure: Build preference stores, metadata filters, tenant-isolated vector indexing, and user deletion tools[cite: 2].

8. **Phase 8** — Spend & Procurement Modules: Add expense logging, receipt parsing, vendor comparison, and purchase request approvals[cite: 2].

9. **Phase 9** — Proactive Automation Engine: Add cron-based triggers for scheduled daily preparation briefings and renewal alerts[cite: 2].

10. **Phase 10** — Enterprise Hardening: Implement SAML/OIDC SSO, tenant workspace isolation, secrets rotation, and audit logging[cite: 2].

11. **Phase 11** — Adversarial Evaluation & Release: Run prompt-injection suites, safety tests, state recovery tests, and benchmark SLAs[cite: 2].


## 8. Runnable Starter Blueprint (Google ADK + Python)

### Project Repository Structure

```text
atlas-core/
├── config.py
├── main.py
├── agents/
│   ├── __init__.py
│   ├── orchestrator.py
│   ├── communications.py
│   └── calendar.py
├── tools/
│   ├── __init__.py
│   ├── calendar_tools.py
│   └── mail_tools.py
├── requirements.txt
└── README.md

```


