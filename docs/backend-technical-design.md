# Backend Technical Design — PropVista CRM

> **Doc 5 of 9** in the new documentation set requested for this project.
> Built from `00-project-overview.md` and `01-prd.md`.
>
> **Status:** Draft v1.1 · **Last updated:** July 2026
> **Revision note:** v1.1 removes multi-tenancy. There is no `tenant_id`, no Row-Level Security policy scoping by tenant, no Tenant Service, and no super-admin surface. Access control is now purely role-based (`customer`, `agent`, `admin`) over one shared database.

---

## 1. Purpose & Scope

This document specifies the **backend architecture**: how the single FastAPI service is structured, how it's organized into services/modules, how it talks to Supabase and Amazon Bedrock, and how background work is scheduled.

---

## 2. Architecture Overview

The backend is a single FastAPI application serving both the customer-facing public API and the admin API from one codebase, split by route prefix and role-based permission rather than by separate deployments. It is async throughout, since chat responses stream and AI calls run concurrently with other request handling.

```
┌────────────────────────────────────────────────────────────┐
│                      FastAPI Application                     │
│                                                                │
│         ┌──────────────┐        ┌──────────────┐             │
│         │  Public API   │        │  Admin API    │             │
│         │  routers      │        │  routers      │             │
│         └──────┬───────┘        └──────┬───────┘             │
│                └──────────────────┼─────────────────┘         │
│                                    │                           │
│                          ┌─────────▼─────────┐                 │
│                          │   Service layer     │                 │
│                          │  (business logic)   │                 │
│                          └─────────┬─────────┘                 │
│                                    │                           │
│         ┌────────────────────┼────────────────────┐            │
│         │                    │                     │            │
│   ┌─────▼──────┐    ┌────────▼────────┐   ┌────────▼───────┐   │
│   │ Data access │    │  AI orchestration │   │  Jobs / worker │   │
│   │ (SQLAlchemy)│    │  (Bedrock client)  │   │  (pg_cron +    │   │
│   │             │    │                    │   │   Python poll) │   │
│   └─────┬──────┘    └────────┬────────┘   └────────┬───────┘   │
└─────────┼───────────────────┼──────────────────────┼───────────┘
          │                    │                      │
   ┌──────▼──────┐     ┌───────▼───────┐      ┌───────▼───────┐
   │  Supabase    │     │  Amazon Bedrock│      │  SendGrid /    │
   │  (Postgres,  │     │  (Claude models)│      │  Twilio        │
   │  Auth,       │     │                 │      │                │
   │  Storage)    │     │                 │      │                │
   └─────────────┘     └────────────────┘      └────────────────┘
```

**Layering rules:**
- Routers only parse/validate the request and call a service method — no business logic in a router.
- Services own business logic and call the data-access layer and/or AI orchestration layer; services never talk to the database directly.
- The data-access layer is the only place raw SQLAlchemy sessions are used.
- The AI orchestration layer is the only place Bedrock is called from — no router or unrelated service calls Bedrock directly, so model/prompt changes have one place to land.

---

## 3. Folder / Module Structure (Proposed)

```
backend/
  app/
    main.py                  # FastAPI app factory, middleware, router mounting
    api/
      public/                # customer-facing routers (search, chat, properties, leads, auth-adjacent)
      admin/                 # admin-portal routers (leads, properties, users, ai-config, reports, notifications, branding)
    services/
      property_service.py
      search_service.py
      chatbot_service.py
      recommendation_service.py
      lead_service.py
      notification_service.py
      metrics_service.py       # the single source of truth for conversion rate etc.
      cms_service.py
      user_service.py
    ai/
      bedrock_client.py         # thin wrapper around Bedrock invoke/streaming calls
      prompts/                  # per-feature prompt templates, versioned
      tools/                    # tool definitions for chatbot function-calling
      embeddings.py              # embedding generation for property content
    data/
      models/                   # SQLAlchemy models
      repositories/              # query logic per entity
      migrations/                # Alembic
    jobs/
      worker.py                  # polls the jobs table
      handlers/                  # one handler per job type (embed_property, bulk_import, dispatch_notification, mark_stale_leads)
    core/
      config.py                  # settings, per-environment
      auth.py                    # Supabase Auth integration, role/permission checks
      errors.py
  tests/
```

---

## 4. Access Control

There is one shared database and no data-isolation boundary to enforce beyond normal per-record ownership (e.g. a customer only sees their own saved properties and inquiries; an agent sees leads assigned to them plus the shared unclaimed queue). Enforcement is role-based, checked server-side on every request:

- **Role check:** every admin-portal route validates the caller's role (`agent` or `admin`) before executing; a restricted-role user cannot call an endpoint outside their permission set regardless of what the UI shows (NFR-SEC1).
- **Ownership check:** where a record has an owner (a lead, a saved profile), the service layer confirms the caller either owns it or holds a role that grants broader access (e.g. `admin` can see all leads; `agent` can see their own plus the shared queue).
- **Public routes:** the public API (search, listings, chat) requires no authentication; session-scoped state (favorites, chat history) is keyed to a session ID until a user registers/logs in, at which point it migrates to their account.

`core/auth.py` resolves the caller's identity and role from Supabase Auth once per request and makes it available to every downstream service call via FastAPI's dependency injection.

---

## 5. Core Services

### 5.1 Property Service
Owns listing CRUD, status transitions (Draft → Pending Approval → Published; Featured/Sold/On Hold/Archived flags), and bulk upload validation (FR9.1–FR9.4). On publish, it enqueues an `embed_property` job rather than calling the AI orchestration layer synchronously — so a save request returns immediately regardless of embedding latency (FR9.5).

### 5.2 Search Service
Accepts a free-text query, calls the AI orchestration layer to parse it into structured constraints plus a semantic component, combines that with the structured filter query, and ranks results (FR2.1–FR2.3). Falls back to a filter-only path if the AI parse step errors or exceeds its timeout budget (FR2.6) — this fallback is implemented as a try/except around the AI call, not a separate user-facing mode the frontend has to detect.

### 5.3 Chatbot Service
Manages conversation state, streams responses back through the API layer, and exposes backend tool calls (property lookup, lead creation, callback scheduling) to the model so it answers from live data rather than free generation (FR1.3). Escalation is a state flag on the conversation, checked by the admin's Conversation Log Viewer and any real-time notification path.

### 5.4 Recommendation Service
Takes wizard answers, treats a skipped step as "no constraint" and renormalizes remaining criterion weights rather than scoring the missing criterion as zero (FR3.1). Scoring produces both a percentage and the plain-language reason lines the frontend renders (FR3.2). Admin-level weighting overrides (FR3.5, FR13.3) are read from configuration before scoring.

### 5.5 Lead Service
Creates leads from every capture channel with a mandatory, non-null source (FR10.4). Owns the atomic claim operation (FR10.2a) — implemented as a single conditional update (claim succeeds only if `owner_id IS NULL`), not a read-then-write, to avoid the race a naive implementation would have. Enforces the fixed pipeline stage enum (FR10.1).

### 5.6 Notification Service
Does not send anything synchronously from the request path. Every notification-worthy event enqueues a job; the jobs worker dispatches it. This is what guarantees a SendGrid/Twilio outage can't block the action that triggered it (NFR-R1). Batches per-recipient (FR3.4) — a bulk import triggering 200 matches produces one queued notification per affected recipient, not 200.

### 5.7 Metrics Service
The single computation point for conversion rate, active leads, and response time (NFR-M1). Dashboard, Agent Leaderboard, and Reports all call this service rather than each computing their own aggregation — this is a design constraint, not just a convention, since three independently-computed "conversion rates" is the fastest way to make a CRM's numbers distrusted.

### 5.8 CMS Service
CRUD for pages, banners, and blog posts, plus SEO metadata fields (FR14.1, FR14.2). Serves both the admin write path and (pending the API Spec resolving the currently-missing public read endpoint, per `00-project-overview.md` §9.1) the public read path.

### 5.9 User & Auth
Wraps Supabase Auth for identity, but keeps the application's own `users` table as the source of truth for role — so a role change takes effect immediately rather than waiting for a JWT to expire and refresh (per `00-project-overview.md` §9, confirmed decision). Enforces the three fixed roles (FR11.1).

---

## 6. AI Orchestration Layer

### 6.1 Bedrock Client
A thin wrapper (`ai/bedrock_client.py`) around Bedrock's invoke and streaming invoke calls. It is the only code path that talks to Bedrock — every service above calls into it rather than importing a Bedrock SDK directly, so model version changes, prompt changes, and retry/timeout policy live in one place.

### 6.2 Model Selection Per Task
Per `00-project-overview.md` §9, the specific Claude model per task (a lighter/faster model for search-query parsing vs. a stronger model for chatbot conversation and recommendation reasoning) is an open decision, to be finalized alongside cost/latency tradeoffs. This design reserves a per-task model configuration (in `core/config.py`) so that decision doesn't require a code change to apply once made.

### 6.3 Tool Calling
The chatbot uses Bedrock's tool-calling support to invoke backend functions (property lookup, lead creation, visit scheduling) rather than generating those actions as free text — this is what FR1.3 means by "not hallucinated answers." Tool definitions live in `ai/tools/`, each mapping to a specific service method with its own input validation, so a tool call is treated with the same rigor as a direct API request.

### 6.4 Prompt & Config Management
Chatbot greeting, FAQ content, and escalation rules (FR13.1) are admin-configured content that gets injected into the system prompt at request time, not baked into a static prompt file — this is what lets an admin's AI Configuration changes take effect for new conversations without a deployment (NFR-U4).

### 6.5 Embeddings
Property content is embedded (model choice per `00-project-overview.md` §9: Amazon Titan Text Embeddings V2, 1024 dimensions, via Bedrock) as part of the `embed_property` background job, not the publish request itself. A failed embedding job must alert rather than fail silently (NFR-R2) — the property would otherwise be live and simply invisible to AI search with no error surfaced anywhere.

---

## 7. Background Jobs & Scheduling

Background work runs on **pg_cron** (SQL-only scheduled jobs) plus a **Python worker polling a `jobs` table**, per the confirmed decision in `01-prd.md` §18. Job types include: `embed_property`, `bulk_import`, `dispatch_notification`, and `mark_stale_leads`.

- `mark_stale_leads` runs on a schedule (pg_cron) and enqueues `dispatch_notification` jobs for any lead unclaimed past the configured threshold — this is what makes FR10.2b's mandatory stale-lead alert actually fire.
- `dispatch_notification` is consumed by the Python worker, which calls SendGrid/Twilio/in-app delivery and records dispatch status per recipient/channel, independent of the request that originally triggered the notification.
- Failed jobs are recorded with enough detail to alert an operator — silent job failure is explicitly called out as unacceptable for the embedding case (NFR-R2) and applies as a general principle to every job type.

---

## 8. Cross-Cutting Concerns

### 8.1 Idempotency
Lead-creation endpoints accept an `Idempotency-Key` header; the Lead Service checks for an existing lead with that key before creating a new one, so retried submissions (double-click, network retry) never create duplicate leads (FR6.4). This is a distinct concern from claim atomicity (FR10.2a), which is about concurrent claims, not duplicate creation.

### 8.2 Audit Logging
Every admin action — including denied ones — is written to the append-only audit log (FR11.3). This is implemented as a decorator/middleware on admin routes rather than scattered manual log calls, so a new admin endpoint can't accidentally ship without audit coverage.

### 8.3 Error Handling & Degradation
AI-dependent endpoints (search, chat, recommendation) wrap their Bedrock calls with explicit timeout and fallback handling rather than letting a slow or failed model call surface as a generic 500 — search degrades to filter-only (FR2.6); a chatbot failure should escalate rather than error out silently (FR1.7).

### 8.4 Rate Limiting & Cost Control
SMS dispatch is capped at the Notification Service level (FR15.2) — not just a UI-side warning — since a misconfigured rule is a real, unapproved bill.

---

## 9. Non-Functional Traceability

| NFR (from SRS) | Backend mechanism |
|---|---|
| NFR-SEC1 (role-based access) | Server-side role and ownership checks, §4 |
| NFR-R1 (dispatch failure isolation) | Notification Service is fully job-queued, §5.6, §7 |
| NFR-R2 (silent embedding failure) | Job failure alerting, §6.5, §7 |
| NFR-M1 (one metrics definition) | Metrics Service, §5.7 |
| NFR-U4 (AI config, no deploy) | Prompt/config injected at request time, §6.4 |
| NFR-A1/A2 (audit log) | Audit middleware, §8.2 |

---

## 10. Open Questions

- Final per-task Claude model assignment (§6.2) — pending cost/latency benchmarking.
- Exact stale-lead threshold default.
- Whether the public CMS read path (§5.8) is served by the same routers as admin CMS or a separate public router — depends on the API Specification resolving the missing endpoint.

---

**Next document:** Database Design Document — schema, ER diagrams, and relationships.
