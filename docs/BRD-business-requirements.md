# Business Requirements Document (BRD) — PropVista CRM

> **Doc 1 of 9** in the new documentation set requested for this project.
> Built from `00-project-overview.md` and `01-prd.md`.
>
> **Status:** Draft v1.1 · **Last updated:** July 2026
> **Revision note:** v1.1 removes the multi-tenant SaaS framing present in the original source docs. PropVista CRM is now specified as a **single-business platform** — one real estate business, one public site, one admin portal, one database. Anything in the source docs that was about isolating or provisioning *other* businesses (tenant onboarding, tenant branding, tenant suspension, a platform-level super admin) has been removed rather than adapted, since it doesn't apply to a single-business system.

---

## 1. Purpose of This Document

This BRD captures the **business case** for PropVista CRM — why it's being built, who it's for, and what business outcomes it must produce. It stays out of technical implementation detail (that lives in the Backend Technical Design, Database Design, and API Specification documents that follow). Engineering-facing functional requirements live in the SRS.

---

## 2. Business Vision

PropVista CRM is an **AI-first real estate CRM** for a single real estate business — a brokerage, developer, or agency. It gives that business a public website *and* an admin CRM in one product, differentiated by three AI capabilities — conversational chat, natural-language search, and guided requirement-matching — that reduce the effort a property seeker spends finding a home and the effort the business spends qualifying and converting that seeker into a closed deal.

The platform serves two audiences:

- **Property seekers** (customers), who currently rely on manual filter-based search and cold contact forms on most real estate sites.
- **The business itself** (owner/admin and sales agents), who currently manage leads through spreadsheets, generic CRMs not built for property, or no system at all.

---

## 3. Business Objectives

| # | Objective | Why it matters |
|---|---|---|
| BO1 | Reduce time-to-first-relevant-result for a property seeker | Manual filter search is the industry default and loses visitors who don't know how to express their needs as filters |
| BO2 | Increase the percentage of site visitors who convert into a tracked lead | Every visitor interaction (chat, search, recommendation) is an opportunity to capture intent; today most of that intent leaves without a trace |
| BO3 | Give agents a working queue instead of a spreadsheet | A lead with no defined owner and no stage tracking is a lead that gets forgotten |
| BO4 | Keep the AI capabilities genuinely useful, not a gimmick | Chat, search, and recommendation only earn their place if they measurably out-convert plain filter search |

---

## 4. In-Scope Business Processes (MVP)

### 4.1 Customer-facing processes
- Discover properties via natural-language search or a conversational assistant, instead of only manual filters.
- Complete a guided requirement questionnaire and receive a ranked, explainable shortlist.
- Browse listings, view full property detail, and request a callback or site visit.
- Optionally register to save searches, save properties, and track inquiry history.

### 4.2 Business-facing (admin/agent) processes
- Publish and manage property listings, including bulk upload and an optional approval workflow.
- Receive leads from every source (chat, search, requirement form, contact form, property page, walk-in) into one pipeline.
- Claim leads from a shared queue and move them through a fixed pipeline to a won or lost outcome.
- Configure the AI chatbot, search, and recommendation behavior without engineering involvement.
- Manage listing content, homepage banners, and static pages without a developer.
- Report on leads, sales, and inventory; receive notifications on key events (new lead, stale lead, escalation, listing expiring).

---

## 5. Out of Scope (This Phase)

- Payment/subscription billing flows.
- Third-party listing portal syndication (auto-posting to external portals).
- Native mobile apps (mobile-responsive web only).
- Advanced BI / data-warehouse-style reporting (basic reports only).
- Two-factor authentication on the admin portal (accepted risk at MVP).

---

## 6. Stakeholders

| Stakeholder | Interest |
|---|---|
| Property seeker (visitor / registered customer) | Fast, relevant, low-effort property discovery |
| Sales agent | A working lead queue and clear ownership; credit for performance |
| Admin / business owner | A live, working site with minimal setup effort; visibility into pipeline health; full control over listings, team, and AI configuration |
| Engineering | A requirements set that is unambiguous, complete, and doesn't require re-litigating decisions mid-build |

---

## 7. High-Level Business Process Flows

### 7.1 Customer discovery-to-lead flow

A visitor arrives at the site with no login required. They discover properties through AI search, the AI chatbot, or the guided AI recommendation flow (or plain browsing). Any of these paths can end in a lead: the chatbot can capture contact info mid-conversation, a property detail page has a persistent callback/visit CTA, and the recommendation flow's shortlist links back into the same contact paths. A lead is created the moment intent is captured, tagged with its originating source, and enters the CRM pipeline unassigned.

### 7.2 Lead-to-close flow

A new lead lands in a shared queue that every agent can see. An agent claims it, and from that point it is theirs to work: notes, call logs, follow-up reminders, and stage moves (Contacted → Site Visit Scheduled → Negotiation → Closed Won/Lost) are all logged with a timestamp. If a lead sits unclaimed past a configured threshold, a stale-lead alert fires — this is the safety net for a queue with no automatic assignment.

### 7.3 AI configuration feedback loop

Everything a customer experiences through chat, search, and recommendation is governed by settings the admin controls — greeting scripts, FAQ content, escalation rules, and criteria weighting. Changes here take effect immediately, without a deployment, so the business can tune the AI experience the way they'd tune a marketing campaign.

---

## 8. Success Metrics (Business-Level)

| Metric | What it tells the business |
|---|---|
| Conversion rate (closed won ÷ closed leads) | Whether the pipeline is actually producing deals |
| Unclaimed lead count | Whether leads are being worked or lost in the shared queue |
| Sessions (self-hosted counter) | Site traffic, without third-party analytics dependency |
| Property views over time | Which listings are attracting attention |
| Agent-level: leads closed, response time, conversion | Individual performance and coaching signal |

---

## 9. Key Business Decisions Already Made

- **No automatic lead assignment** — leads are claimed manually from a shared queue. This was a deliberate choice over round-robin, backstopped by the mandatory stale-lead alert.
- **Fixed CRM pipeline stages** — configurable stage names are out of scope for MVP.
- **Match scores are shown to customers as a raw percentage** in both AI search and AI recommendation results.
- **SMS notifications are opt-in and capped**, and require regulatory registration (DLT) in India — a real lead-time item for go-live planning, not just a technical toggle.

---

## 10. Assumptions

- This is a single-business deployment; there is no requirement to isolate data between separate businesses or to onboard other businesses onto shared infrastructure.
- The business is expected to provide its own property content; the platform does not source or scrape listings on its behalf.
- English-language and India-market assumptions are embedded in some decisions (e.g. SMS/DLT); expansion to other markets is not assumed in this phase.

---

## 11. Risks (Business-Level)

| Risk | Business impact |
|---|---|
| No 2FA on admin portal | A single phished admin password exposes the full customer database |
| Manual claim can leave leads ownerless | Mitigated by mandatory stale-lead alerting, but depends on it actually being configured |

---

## 12. Glossary

| Term | Meaning |
|---|---|
| Lead | A tracked record of buyer/renter intent, created from any contact channel |
| Claim | The act of an agent taking ownership of an unassigned lead |
| Match score | The AI-generated percentage indicating how well a property fits a stated or inferred requirement |
| Stale lead | A lead that has sat unclaimed (or unworked) past a configured time threshold |

---

**Next document:** SRS — detailed functional and non-functional requirements, building on this business context.
