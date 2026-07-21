# Admin Portal Functional Specification — PropVista CRM

> **Doc 4 of 9** in the new documentation set requested for this project.
> Built from `00-project-overview.md` and `01-prd.md`.
>
> **Status:** Draft v1.1 · **Last updated:** July 2026
> **Revision note:** v1.1 removes multi-tenancy. The Super Admin — Tenant Management screen is removed entirely (there is only one business, so there's nothing to provision or suspend). Tenant Branding is renamed Branding Settings and now configures the one business's site directly. All "per tenant" language elsewhere is replaced with plain admin-level configuration.
>
> **Note on APIs in this document:** endpoint paths below are proposed to keep this spec buildable; they are not yet ratified. The **API Specification** document (doc 7 of 9) is the source of truth once written.

---

## 1. Purpose & Scope

This document specifies every screen in the **admin portal** — the React application used by agents and the admin/owner. It is desktop-first (per SRS NFR-U2); the one documented mobile exception is noted at each relevant screen. Screens map back to Modules 8–16 in `01-prd.md`.

---

## 2. Screen Inventory

| # | Screen | Role(s) | Module |
|---|---|---|---|
| 1 | Admin Dashboard | Agent, Admin | Module 8 |
| 2 | Property List & Detail | Agent, Admin | Module 9 |
| 3 | Bulk Upload | Admin | Module 9 |
| 4 | Approval Queue | Admin | Module 9 |
| 5 | Lead Pipeline — Kanban | Agent, Admin | Module 10 |
| 6 | Lead Pipeline — Table | Agent, Admin | Module 10 |
| 7 | Lead Detail | Agent, Admin | Module 10 |
| 8 | Users & Roles | Admin | Module 11 |
| 9 | Audit Log | Admin | Module 11 |
| 10 | Agents | Admin | Module 12 |
| 11 | Agent Leaderboard | Agent, Admin | Module 12 |
| 12 | AI Configuration — Chatbot | Admin | Module 13 |
| 13 | AI Configuration — Search | Admin | Module 13 |
| 14 | AI Configuration — Recommendation Weighting | Admin | Module 13 |
| 15 | Conversation Log Viewer | Admin | Module 13 |
| 16 | CMS — Pages & Banners | Admin | Module 14 |
| 17 | Reports | Admin | Module 15 |
| 18 | Notification Rules | Admin | Module 15 |
| 19 | Branding Settings | Admin | Module 16 |

---

## 3. Screen Specifications

### 3.1 Admin Dashboard
**Purpose:** Landing screen after login; operational snapshot.

**Key components:**
- KPI cards: total listings, active leads, conversion rate, **Sessions** (self-hosted counter — must be labelled "Sessions", never "Visitors", since it counts sessions not people) (FR8.1).
- Charts: lead source breakdown, property views over time (FR8.2).
- Recent activity feed: new leads, listing changes, chatbot escalations (FR8.3).
- **Unclaimed lead count**, surfaced prominently — not buried in a chart — since under manual claim (FR10.2) this is the number that tells the owner whether leads are being lost (FR8.4).

**States & edge cases:** All metrics must reconcile with the Reports module for the same date range — both call the shared `metrics_service`, never two independent calculations.

**APIs used:** `GET /api/v1/admin/dashboard/kpis`, `GET /api/v1/admin/dashboard/activity`.

**Related FRs:** FR8.1–FR8.4.

---

### 3.2 Property List & Detail
**Purpose:** CRUD for listings, including media management.

**Key components:** List view with filters (status, type); detail/edit form covering photos, video, floor plans, documents (FR9.1); status flags — Featured, Sold, On Hold, Archived (FR9.4).

**Workflow:** On save, the listing is automatically queued for embedding/indexing for AI Search (FR9.5) via the background jobs mechanism, not synchronously in the save request, so the admin sees an immediate save confirmation while indexing completes shortly after.

**States & edge cases:** If approval is required (FR9.3), a non-admin save routes the listing to Pending Approval rather than publishing directly — the UI must clearly show which state the listing is in and why it isn't live.

**APIs used:** `GET/POST/PUT/DELETE /api/v1/admin/properties`, `POST /api/v1/admin/properties/{id}/reject`, `POST /api/v1/admin/properties/bulk-status`.

**Related FRs:** FR9.1, FR9.3, FR9.4, FR9.5.

---

### 3.3 Bulk Upload
**Purpose:** CSV/Excel import of many listings at once.

**Key components:** File upload, column-mapping step, validation results table (row-level pass/fail with reasons), commit action.

**Workflow:** Upload creates a `bulk_upload` job; the frontend polls or subscribes for progress. Row-level errors are shown per row, not just a single pass/fail for the whole file (FR9.2).

**States & edge cases:** A partially valid file should let the admin commit the valid rows while seeing exactly which rows failed and why — not an all-or-nothing import.

**APIs used:** `POST /api/v1/admin/properties/bulk-upload`, `GET /api/v1/admin/properties/bulk-upload/{job_id}`.

**Related FRs:** FR9.2.

---

### 3.4 Approval Queue
**Purpose:** Review listings sitting in Pending Approval (only relevant if FR9.3 approval is enabled).

**Key components:** Queue list, side-by-side listing preview, approve/reject actions with a reject reason field.

**APIs used:** `GET /api/v1/admin/properties?status=pending_approval`, `POST /api/v1/admin/properties/{id}/approve`, `POST /api/v1/admin/properties/{id}/reject`.

**Related FRs:** FR9.3.

---

### 3.5 Lead Pipeline — Kanban
**Purpose:** Primary working view for agents; visualizes the fixed pipeline.

**Key components:** Fixed columns — New → Contacted → Site Visit Scheduled → Negotiation → Closed Won / Closed Lost (FR10.1, no configurable stage names at MVP). The **New** column is shared and visible to every agent (FR10.2); dragging a card out of New is how an agent claims it.

**Workflow:**
1. A new lead appears in the shared New column for every agent to see.
2. An agent drags it into Contacted (or another stage) to claim it — this drag action is the claim.
3. The claim call must be atomic server-side: if two agents drag the same card near-simultaneously, exactly one succeeds and the other sees the card already gone/reassigned, with a clear (not silent) UI message (FR10.2a).
4. Admins can drag/reassign any lead regardless of current owner, at any time.

**States & edge cases:** This screen does not have a required mobile layout — per NFR-U2 it falls back to the Table view (§3.6) below a defined breakpoint, since drag-and-drop Kanban is not a phone-usable pattern.

**APIs used:** `GET /api/v1/admin/leads?view=kanban`, `POST /api/v1/admin/leads/{id}/claim`, `POST /api/v1/admin/leads/{id}/stage`.

**Related FRs:** FR10.1, FR10.2, FR10.2a.

---

### 3.6 Lead Pipeline — Table
**Purpose:** Non-drag alternative view, and the mobile fallback for §3.5.

**Key components:** Sortable columns including **unclaimed age** — the sort that surfaces a rotting queue at a glance (FR10.5).

**APIs used:** `GET /api/v1/admin/leads?view=table&sort=unclaimed_age`.

**Related FRs:** FR10.5.

---

### 3.7 Lead Detail
**Purpose:** Full record for a single lead — the agent's working surface once claimed.

**Key components:** Contact info, source (chatbot / AI search inquiry / requirement form / contact form / property details page / walk-in — manually added) (FR10.4), notes, call logs, follow-up reminders (FR10.3), stage history with timestamps.

**States & edge cases:** Every lead must show a non-null source, even walk-ins entered manually by an agent — "unknown" is not an acceptable source value.

**APIs used:** `GET /api/v1/admin/leads/{id}`, `POST /api/v1/admin/leads/{id}/notes`, `POST /api/v1/admin/leads/{id}/follow-up`.

**Related FRs:** FR10.3, FR10.4.

---

### 3.8 Users & Roles
**Purpose:** Manage admin-portal accounts.

**Key components:** User list; invite flow (email invite, no self-registration for admin-portal roles) (FR11.2); role assignment limited to the three fixed roles (FR11.1) — `customer`, `agent`, `admin`.

**States & edge cases:** A role change must take effect immediately, without requiring the affected user to re-log in — the frontend should not cache stale permissions client-side beyond the current request.

**APIs used:** `GET /api/v1/admin/users`, `POST /api/v1/admin/users/invite`, `POST /api/v1/admin/users/{id}/resend-invite`, `POST /api/v1/admin/users/{id}/revoke-invite`, `PUT /api/v1/admin/users/{id}/role`.

**Related FRs:** FR11.1, FR11.2.

> **Open dependency:** resend/revoke invite endpoints are listed in `00-project-overview.md` §9.1 as not yet in the API spec — proposed above pending that document.

---

### 3.9 Audit Log
**Purpose:** Read-only, append-only record of admin actions — including denied ones.

**Key components:** Filterable table: actor, action, entity, timestamp, allowed/denied. No edit or delete affordances anywhere in the UI (the backend enforces this too, but the UI shouldn't even offer it) (FR11.3).

**APIs used:** `GET /api/v1/admin/audit-log`.

**Related FRs:** FR11.3.

---

### 3.10 Agents
**Purpose:** Manage agent profiles and see their workload.

**Key components:** Profile (contact info), assigned listings, assigned leads (FR12.1).

**APIs used:** `GET /api/v1/admin/agents`, `GET /api/v1/admin/agents/{id}`.

**Related FRs:** FR12.1.

---

### 3.11 Agent Leaderboard
**Purpose:** Performance comparison across agents.

**Key components:** Leads closed, response time, conversion rate per agent (FR12.2), ranked leaderboard (FR12.3). All figures come from the shared `metrics_service` — same conversion-rate definition as the Dashboard and Reports.

**States & edge cases:** Metrics must recompute correctly as leads change stage or owner mid-period — a lead reassigned from Agent A to Agent B shouldn't double-count or vanish from either agent's numbers depending on when the period boundary falls.

**APIs used:** `GET /api/v1/admin/agents/leaderboard`.

**Related FRs:** FR12.2, FR12.3.

---

### 3.12 AI Configuration — Chatbot
**Purpose:** Admin control of chatbot behavior, no engineering involvement required.

**Key components:** Greeting script editor, FAQ library (question/answer pairs), escalation rule builder (e.g. "escalate after N failed resolutions" or on explicit keyword) (FR13.1).

**States & edge cases:** Saved changes apply to new conversations immediately — no deployment step, and no impact on conversations already in progress at save time.

**APIs used:** `GET/PUT /api/v1/admin/ai-config/chatbot`.

**Related FRs:** FR13.1.

---

### 3.13 AI Configuration — Search
**Purpose:** Visibility into how AI Search is performing.

**Key components:** Table/list of queries with no or low results, so the admin can see what buyers are searching for that isn't matching inventory (FR13.4).

**APIs used:** `GET /api/v1/admin/ai-config/search-insights`.

**Related FRs:** FR13.4.

---

### 3.14 AI Configuration — Recommendation Weighting
**Purpose:** Control of how strongly each requirement criterion (budget, location, amenities, etc.) influences the match score.

**Key components:** Weight sliders or numeric inputs per criterion, with a preview of how a sample profile's shortlist changes (FR13.3, FR3.5).

> **Open dependency:** a weights-preview endpoint is listed in `00-project-overview.md` §9.1 as not yet specified — this screen's preview interaction depends on it.

**APIs used:** `GET/PUT /api/v1/admin/ai-config/recommendation-weights`.

**Related FRs:** FR13.3, FR3.5.

---

### 3.15 Conversation Log Viewer
**Purpose:** Review chatbot conversations, flag problematic ones for follow-up (FR13.2).

**Key components:** Conversation list with flag toggle; full transcript view. Per `01-prd.md` §18a, an agent's own leads' transcripts are visible to that agent, and colleagues' leads are read-only — this screen enforces that scoping, not just the admin's full view.

**APIs used:** `GET /api/v1/admin/chat/conversations`, `POST /api/v1/admin/chat/conversations/{id}/flag`.

**Related FRs:** FR13.2, FR1.8.

---

### 3.16 CMS — Pages & Banners
**Purpose:** Edit homepage banners, blog posts, and static pages without developer involvement (FR14.1), with SEO fields per page (FR14.2).

**Key components:** Page/banner list, rich content editor, SEO metadata fields (title, meta description, slug), draft/publish toggle.

**APIs used:** `GET/POST/PUT /api/v1/admin/cms/pages`, `GET/POST/PUT /api/v1/admin/cms/banners`.

**Related FRs:** FR14.1, FR14.2.

---

### 3.17 Reports
**Purpose:** Leads, sales, and inventory reporting with filters and export.

**Key components:** Filter builder (date range, source, agent, property type); report preview table; export to PDF/Excel (FR15.1). No "saved report" entity — export re-runs the same query with the same parameters rather than reading a cached result (FR15.1 decision). A sales report reads `leads.deal_value`, set at `closed_won`, not the property's asking price (FR15.1a).

**States & edge cases:** If a requested date range exceeds the row cap, the UI must tell the admin to narrow the range — never silently return a truncated result set (FR15.1).

**APIs used:** `POST /api/v1/admin/reports/leads`, `POST /api/v1/admin/reports/sales`, `POST /api/v1/admin/reports/inventory`, each with an `?export=pdf|xlsx` variant.

**Related FRs:** FR15.1, FR15.1a.

---

### 3.18 Notification Rules
**Purpose:** Configure event → channel → recipient mappings (FR15.2).

**Key components:** Rule list; per-rule event selector (`new_lead`, `lead_escalated`, `lead_stale`, `lead_assigned`, `follow_up_due`, `property_pending_approval`, `listing_expiring`); channel checkboxes (in-app, email, SMS); recipient selector.

**States & edge cases:**
- The **`lead_stale`** rule is mandatory, not optional (FR10.2b dependency) — the UI should not let an admin fully disable it, only adjust its threshold, since manual claim loses leads without it.
- SMS is opt-in and capped; enabling it surfaces the DLT registration requirement for Indian numbers before it can go live.
- A provider outage (SendGrid/Twilio) must never be presented as blocking any other action in the admin UI — dispatch failures show up as a status on the notification itself, not as an error on lead creation or any other trigger.

**APIs used:** `GET/POST/PUT /api/v1/admin/notification-rules`.

**Related FRs:** FR15.2, FR10.2b.

---

### 3.19 Branding Settings
**Purpose:** Configure logo, colors, custom domain with live preview (FR16.1).

> **Status: post-MVP.** Per `01-prd.md` §17, MVP ships one fixed palette — this screen's backing columns and endpoint are specced but not built in the MVP sprints. This section documents the target design so it isn't re-discovered later.

**Key components:** Logo upload, primary/accent color pickers, custom domain field with a DNS verification status indicator (a domain must be verified before it is routed — an unverified domain is a hijacking vector).

**Constraint:** the admin may override `primary` and its derived ramp only. The `tertiary` family is the platform's AI-intelligence signal and stays fixed; the color picker UI must not expose a way to apply a custom color to `tertiary`.

**APIs used:** `PUT /api/v1/admin/branding` *(specced, not active pre-MVP)*.

**Related FRs:** FR16.1.

---

## 4. Cross-Cutting Notes

- **Desktop-first:** All screens above target desktop; only the Lead Pipeline explicitly defines a mobile fallback (table view). Other admin screens are not required to have a bespoke mobile layout at MVP.
- **Permission enforcement:** Every screen's actions must be enforced server-side per role — the UI hiding a button is a usability courtesy, not the security boundary.
- **Shared metrics:** Dashboard (§3.1), Agent Leaderboard (§3.11), and Reports (§3.17) must never show conflicting numbers for the same date range — all three call the one `metrics_service`.

---

## 5. Open Questions

- Exact shape of the recommendation-weights preview endpoint (§3.14) — not yet specified.
- Resend/revoke invite endpoints (§3.8) — not yet specified.
- Whether AI Search insights (§3.13) needs its own filter/date-range controls or is a fixed recent-window view — needs product confirmation.

---

**Next document:** Backend Technical Design — FastAPI architecture, services, endpoints, business logic, and AI orchestration.
