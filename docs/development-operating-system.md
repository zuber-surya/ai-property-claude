# Development Operating System (DevOS) — PropVista CRM

> **Doc 9 of 9** in the new documentation set requested for this project.
> Built from `00-project-overview.md`, `01-prd.md`, and consistent with docs 1–8 (single-tenant, three roles).
>
> **Status:** Draft v1.0 · **Last updated:** July 2026

---

## 1. Purpose & Scope

This document defines **how the system gets built**: the phase order, what each phase delivers, how to hand each phase to Claude Code, how progress is tracked, and how each phase is tested before moving to the next. It is the execution layer sitting on top of docs 1–8 — it doesn't introduce new requirements, it sequences the ones already specified.

---

## 2. Development Principles

1. **Build the non-AI spine before the AI features.** Properties, leads, and users are the substrate every AI feature reads from or writes to — building them first means AI Search, Chat, and Recommendation have real data and a real CRM to plug into instead of being built against mocks.
2. **Every phase ends in something demoable and testable**, not a partial layer across the whole system. A phase is "done" when its acceptance criteria (from the PRD/SRS) pass, not when the code compiles.
3. **One document set is the spec.** Claude Code prompts for every phase point back at the specific docs in this set (BRD/SRS/functional specs/DB design/API spec/UI-UX spec) rather than re-explaining requirements inline — this keeps the docs and the code from drifting apart.
4. **Test cases trace to acceptance criteria.** Every `TC-*` below maps to an acceptance criterion already stated in `01-prd.md` or the SRS — this plan does not invent new acceptance bars.

---

## 3. Phase Plan Overview

| Phase | Name | Primary modules | Depends on |
|---|---|---|---|
| 0 | Foundation | Auth, base schema, both React app shells | — |
| 1 | Property Core | Module 9 (non-AI parts) | Phase 0 |
| 2 | Lead & CRM Pipeline | Module 10, Module 6 | Phase 1 |
| 3 | Users, Roles, Agents | Module 11, Module 12 | Phase 0 |
| 4 | AI Search | Module 2 | Phase 1 |
| 5 | AI Recommendation | Module 3 | Phase 1, 4 |
| 6 | AI Chatbot | Module 1 | Phase 1, 2 |
| 7 | Customer Portal | Module 7 | Phase 2, 4, 5 |
| 8 | Dashboard, Metrics & Reports | Module 8, Module 15 (reports) | Phase 2, 3 |
| 9 | AI Configuration, Notifications, CMS | Module 13, 14, 15 (notifications) | Phase 4, 5, 6, 8 |
| 10 | Hardening & Launch Prep | Cross-cutting NFRs | All prior |

---

## 4. Phase Detail

### Phase 0 — Foundation
**Delivers:** Supabase project with Auth wired up; base schema for `users`; FastAPI skeleton with the layered structure from the Backend Technical Design (§3); both React app shells (public/customer, admin) with routing per the UI/UX sitemap; CI pipeline running lint + tests on every push.

**Claude Code prompt:**
> "Using `02-backend-technical-design.md` §3 as the folder structure and `06-database-design-document.md` §4.1 for the `users` table, scaffold the FastAPI backend with the app/api/services/data/ai/jobs/core layout. Wire up Supabase Auth per `05-backend-technical-design.md` §4 [access control] — role stored in the app's own `users` table, not trusted from the JWT. Do not implement any feature endpoints yet, just the skeleton, health check, and auth dependency."

**Test cases:**
- `TC-0.1`: A request with a valid JWT resolves to the correct `users.role`.
- `TC-0.2`: A request with an invalid/expired JWT is rejected with `401 unauthenticated`.

**Definition of done:** Both apps build and deploy to a dev environment; a logged-in user's role is visible in the app.

---

### Phase 1 — Property Core
**Delivers:** `properties`, `property_media`, `amenities`, `property_amenities` tables; Property Service (CRUD, status transitions, bulk upload minus AI validation); admin Property List/Detail, Bulk Upload, Approval Queue screens; public Property Listing (filter/sort, no AI yet) and Property Details screens.

**Claude Code prompt:**
> "Implement Module 9 (Property Management) per `01-prd.md` FR9.1–FR9.4 and the Admin Portal Functional Spec §3.2–3.4, using the `properties`/`property_media`/`amenities` schema from the Database Design doc §4.2–4.5 and the endpoints in the API Specification §3.2. Do not implement FR9.5 (embedding/indexing) yet — that's Phase 4. Build the corresponding public Property Listing and Property Details screens per the Customer Portal Functional Spec §4.2–4.3, filter/sort only, no AI search integration yet."

**Test cases:**
- `TC-1.1`: Creating a listing with approval required routes it to Pending Approval, not Published (FR9.3).
- `TC-1.2`: A bulk upload with some invalid rows commits the valid rows and reports errors per row (FR9.2 acceptance criterion).
- `TC-1.3`: Property Details renders correctly with a missing floor plan — layout doesn't break (FR5.1 acceptance criterion).
- `TC-1.4`: Amenities on both the property form and listing filters draw from the same `amenities` table — no free text accepted (FR3.6).

**Definition of done:** An admin can publish a listing; a visitor can browse, filter, and view it end to end, without AI.

---

### Phase 2 — Lead & CRM Pipeline
**Delivers:** `leads`, `lead_notes`, `lead_stage_history` tables; Lead Service including atomic claim; Contact/Callback Modal on the public side; Kanban + Table + Lead Detail on the admin side.

**Claude Code prompt:**
> "Implement Module 10 (Lead & CRM Pipeline) and Module 6 (Contact/Lead Capture) per `01-prd.md` FR10.1–FR10.5 and FR6.1–FR6.4. Use the atomic-claim pattern specified in the Backend Technical Design §5.5 (single conditional UPDATE, not read-then-write) — this is a hard requirement, not an implementation suggestion. Build the Kanban view per the Admin Portal Functional Spec §3.5 with the mobile Table fallback per §3.6. Idempotency-Key handling per the API Specification §1 is required on `POST /leads`."

**Test cases:**
- `TC-2.1`: Two simultaneous claim requests on the same lead result in exactly one owner (FR10.2a) — write this as a concurrency test, not just a sequential one.
- `TC-2.2`: A duplicate lead submission with the same `Idempotency-Key` does not create a second record (FR6.4).
- `TC-2.3`: Every created lead has a non-null `source` (FR10.4) — test every capture channel, including manually-added walk-in.
- `TC-2.4`: The Table view is served instead of Kanban below the mobile breakpoint (NFR-U2).

**Definition of done:** A lead created from any public capture point appears in the shared queue, can be claimed, and can be worked to a closed stage.

---

### Phase 3 — Users, Roles, Agents
**Delivers:** Invite flow, role management, audit log, Agent profile/leaderboard screens.

**Claude Code prompt:**
> "Implement Module 11 (User & Role Management) and Module 12 (Agent Management) per `01-prd.md` FR11.1–FR11.4 and FR12.1–FR12.3, using the three-role model (`customer`/`agent`/`admin`) — there is no `super_admin` in this system (see the revision notes in docs 1, 2, 4, 5). Every admin action, including denied ones, must write to the append-only `audit_log` table per the Database Design doc §4.21; implement this as middleware, not per-route logging calls, per the Backend Technical Design §8.2."

**Test cases:**
- `TC-3.1`: A restricted-role user's call to an out-of-scope admin endpoint returns `403`, and the audit log records it as a denied action (NFR-SEC1, SRS §4.3).
- `TC-3.2`: A role change takes effect on the next request without requiring re-login.
- `TC-3.3`: The audit log table rejects `UPDATE`/`DELETE` at the database privilege level, not just application logic.

**Definition of done:** Full user lifecycle (invite → login → role change → audit trail) works, and agent performance figures compute correctly.

---

### Phase 4 — AI Search
**Delivers:** `embed_property` job, `search_queries` table, Search Service, `POST /search` + autosuggest, Search Bar component with AI/fallback states.

**Claude Code prompt:**
> "Implement Module 2 (AI Search) per `01-prd.md` FR2.1–FR2.6, using the AI orchestration layer from the Backend Technical Design §6. Property embedding runs as the `embed_property` background job (§6.5, §7) — never synchronously on publish. Implement the filter-only fallback (FR2.6) as a try/except around the Bedrock call per §5.2, and make sure it is visibly indicated in the UI per the UI/UX Spec §5.2, not a silent degrade. Log every query to `search_queries` per the Database Design doc §4.11, since AI Configuration's search-insights screen (Phase 9) depends on this data existing from day one."

**Test cases:**
- `TC-4.1`: A representative set of natural-language queries returns results a human reviewer judges relevant (FR2.1 acceptance criterion — requires a curated test query set, build this as part of the phase).
- `TC-4.2`: Forcing a Bedrock timeout falls back to filter-only search without erroring (FR2.6).
- `TC-4.3`: A failed embedding job leaves the property's `embedding_status = failed` and triggers an alert (NFR-R2) — verify the property is flagged, not silently unsearchable.

**Definition of done:** A natural-language query on the live site returns AI-ranked results with visible match scores, and gracefully degrades if the AI call fails.

---

### Phase 5 — AI Recommendation
**Delivers:** `requirement_profiles` tables, Recommendation Service, wizard screen, shortlist rendering, save/edit/delete profile.

**Claude Code prompt:**
> "Implement Module 3 (AI Recommendation) per `01-prd.md` FR3.1–FR3.6, using the Recommendation Service design in the Backend Technical Design §5.4 — a skipped wizard step must renormalize remaining criterion weights, never score the missing criterion as zero. Match scores and ✓/✗ reason lines per FR3.2 render using the API Specification §2.6 response shape. Batched match notifications (FR3.4) are Phase 9's concern (they depend on the notification system) — for this phase, just implement the shortlist generation and profile CRUD."

**Test cases:**
- `TC-5.1`: Skipping a step produces a shortlist scored only on the answered criteria, with weights renormalized (FR3.1).
- `TC-5.2`: A sparse-match scenario still returns a ranked "closest matches" list, never an empty/broken state (FR3.1 acceptance criterion).
- `TC-5.3`: Deleting a saved profile soft-deletes it and a historically-linked lead still resolves correctly (Database Design §4.9).

**Definition of done:** A visitor can complete the wizard and get an explainable shortlist; a registered user can save, edit, and delete their profile.

---

### Phase 6 — AI Chatbot
**Delivers:** `chat_conversations`/`chat_messages` tables, Chatbot Service with tool calling, streaming widget, escalation flow.

**Claude Code prompt:**
> "Implement Module 1 (AI Chatbot) per `01-prd.md` FR1.1–FR1.8, using the tool-calling pattern from the Backend Technical Design §6.3 — the bot must call the Property Service and Lead Service through defined tools, never generate property facts or lead confirmations from free text. Stream responses per the API Specification §2.3 (SSE). Escalation (FR1.7) sets a conversation status flag consumed by Phase 3's audit-adjacent admin view and any real-time notification path built in Phase 9."

**Test cases:**
- `TC-6.1`: Asking the bot about a specific live property returns accurate current data, not a hallucinated answer (FR1.3 acceptance criterion) — verify by comparing bot output against the property record at test time.
- `TC-6.2`: A full conversation ending in contact capture creates a lead with `source = chatbot` (FR1.4, FR1.8 acceptance criterion).
- `TC-6.3`: Explicit escalation request sets the conversation to `escalated` and this is visible to an agent in near-real time (FR1.7 acceptance criterion).

**Definition of done:** The chatbot answers from live data, captures leads, and escalates correctly; conversations are logged and reviewable.

---

### Phase 7 — Customer Portal
**Delivers:** Dashboard, Saved Properties, Requirement Profile view, Inquiry History, Notification Preferences.

**Claude Code prompt:**
> "Implement Module 7 (Customer Portal) per `01-prd.md` FR7.1–FR7.3 and the Customer Portal Functional Spec §4.7–4.11. This phase is mostly composition — Favorites (Phase 1), Requirement Profiles (Phase 5), and Inquiries (Phase 2) already exist; build the aggregated dashboard view and enforce that every query is scoped to the logged-in user only, server-side (FR7.1 acceptance criterion)."

**Test cases:**
- `TC-7.1`: User A cannot retrieve User B's saved properties, inquiries, or requirement profile via direct API calls, not just hidden UI (FR7.1 acceptance criterion).
- `TC-7.2`: Editing notification preferences takes effect on the next batched notification event, not retroactively.

**Definition of done:** A logged-in customer has one coherent home base reflecting everything they've done anonymously and as an account holder.

---

### Phase 8 — Dashboard, Metrics & Reports
**Delivers:** `metrics_service`, Admin Dashboard, Agent Leaderboard (metrics portion), Reports module.

**Claude Code prompt:**
> "Implement the `metrics_service` per the Backend Technical Design §5.7 as the single computation point for conversion rate, active leads, and response time — Dashboard (FR8.1–FR8.4), Agent Leaderboard (FR12.2–FR12.3), and Reports (FR15.1) all call this one service. Implement Reports as synchronous and row-capped per FR15.1 — reject over-cap date ranges per the API Specification §3.10 error shape rather than truncating."

**Test cases:**
- `TC-8.1`: Dashboard and Reports show identical conversion-rate figures for the same date range (FR8.1 acceptance criterion / NFR-M1) — write this as a cross-endpoint consistency test, not two separate unit tests.
- `TC-8.2`: A report request exceeding the row cap returns `422 date_range_too_large`, never a truncated 200.
- `TC-8.3`: `deal_value` on a sales report reflects `leads.deal_value`, not the property's listed price (FR15.1a).

**Definition of done:** All three screens that show metrics agree with each other for any given date range, always.

---

### Phase 9 — AI Configuration, Notifications, CMS
**Delivers:** `jobs`-driven notification dispatch, `notification_rules`, mandatory stale-lead job, AI Configuration screens, CMS pages/banners (admin + public read).

**Claude Code prompt:**
> "Implement Module 13 (AI Configuration), Module 14 (CMS), and the notification half of Module 15 per `01-prd.md` FR13.1–FR13.4, FR14.1–FR14.2, FR15.2. Notification dispatch must be fully job-queued per the Backend Technical Design §5.6/§7 — a SendGrid/Twilio outage must never fail the lead-creation (or any other triggering) request (NFR-R1). Implement `mark_stale_leads` as a pg_cron job per §7, and make the `lead_stale` notification rule non-disableable (only its threshold is editable) per the Admin Portal Functional Spec §3.18. Batch match notifications (FR3.4) — one per recipient per event, not one per matched property."

**Test cases:**
- `TC-9.1`: Simulated SendGrid/Twilio failure does not roll back or error the lead-creation request that triggered the notification (NFR-R1).
- `TC-9.2`: A bulk import of 200 newly-matching properties produces exactly one notification per affected recipient (FR3.4 acceptance criterion).
- `TC-9.3`: Attempting to fully disable the `lead_stale` rule is rejected by the API; adjusting its threshold succeeds (FR10.2b).
- `TC-9.4`: A published CMS page is retrievable from the public read endpoint (closes gap G5) without authentication.

**Definition of done:** Every notification event fires reliably and independently of provider uptime; admins can tune AI behavior and site content without engineering.

---

### Phase 10 — Hardening & Launch Prep
**Delivers:** Security review, accessibility pass (per UI/UX Spec §8), performance/latency budget verification, retention job verification (90d rollups / 24mo chats / 7yr audit), load testing on Search/Chat/Recommendation.

**Claude Code prompt:**
> "Run a hardening pass against the NFRs in the SRS §4: verify tenant-free role-based access control end to end (there's no tenant isolation to test — verify role-based access control does the whole job per NFR-SEC1), confirm the accepted-risk items (no 2FA) are documented rather than silently missing, verify retention jobs exist for the periods in the Database Design doc §5, and load-test `/search`, `/chat/messages`, and `/recommendations` against the latency budgets referenced in NFR-P1."

**Test cases:**
- `TC-10.1`: A full permission matrix test — every role against every admin endpoint — passes with no unexpected `200`s.
- `TC-10.2`: Retention jobs correctly age out data at the documented boundaries without deleting anything still in active use (e.g. a chat conversation tied to an open lead).
- `TC-10.3`: AI-dependent endpoints meet their defined latency budgets under representative load.

**Definition of done:** The system is ready for launch against every NFR in the SRS, with all accepted risks explicitly documented rather than accidentally present.

---

## 5. Testing Strategy

| Layer | Tooling (proposed) | Scope |
|---|---|---|
| Unit | pytest (backend), Vitest/Jest (frontend) | Individual service methods, especially the atomic-claim logic, weight renormalization, and idempotency handling |
| Integration | pytest + test database | API endpoint contracts against the API Specification, including error codes |
| End-to-end | Playwright | The journeys in the UI/UX Spec §4 — one E2E test per journey, minimum |
| Concurrency | Targeted (e.g. pytest-asyncio with simulated parallel requests) | `TC-2.1` (claim race) specifically — this is the one place a naive sequential test would pass while the real system fails |
| Load | k6 or Locust | AI-dependent endpoints only, against NFR-P1 budgets |

Every `TC-*` above traces to an FR or NFR acceptance criterion already defined in `01-prd.md` or the SRS (doc 2) — this plan doesn't introduce test requirements the product docs didn't already imply.

---

## 6. Progress Tracking

A `STATUS.md` file at the repo root tracks phase state, updated at the end of every work session:

```markdown
## Phase status
- [x] Phase 0 — Foundation
- [x] Phase 1 — Property Core
- [ ] Phase 2 — Lead & CRM Pipeline   (in progress: atomic claim implemented, table view pending)
- [ ] Phase 3 — Users, Roles, Agents
...
```

Each phase entry links to its test results (CI run) rather than restating pass/fail manually — the source of truth for "done" is a green CI run against that phase's `TC-*` set, not a checkbox alone.

---

## 7. Global Definition of Done

A phase is done when, and only when:
1. Every `TC-*` listed for that phase passes in CI.
2. Every FR/NFR the phase claims to deliver is demonstrably working against its stated acceptance criterion, not just "implemented."
3. `STATUS.md` is updated.
4. No open item from that phase's Claude Code prompt was silently skipped — if something couldn't be built as specified, it's logged as an open question here or in the owning doc, not left undocumented.

---

## 8. Open Questions

- Concrete latency budgets for NFR-P1 are still "to be defined per feature" — Phase 4/5/6 cannot fully close out `TC-*` load testing until these numbers exist.
- Whether E2E tests run against a seeded staging environment or fully mocked Bedrock responses — mocked is faster and more deterministic for CI, real calls are needed at least once before each phase's demo.
- Curated natural-language test query set for `TC-4.1` needs to be authored — this is content work, not engineering work, and should be scheduled alongside Phase 4 rather than blocking it.

---

This closes the nine-document set: BRD → SRS → Customer Portal Spec → Admin Portal Spec → Backend Technical Design → Database Design → API Specification → UI/UX Specification → DevOS.
