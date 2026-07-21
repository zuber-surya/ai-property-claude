# Software Requirements Specification (SRS) — PropVista CRM

> **Doc 2 of 9** in the new documentation set requested for this project.
> Built from `00-project-overview.md` and `01-prd.md`.
>
> **Status:** Draft v1.1 · **Last updated:** July 2026
> **Revision note:** v1.1 removes multi-tenancy. This is now a single-business system: one public site, one admin portal, one database, three roles (`customer`, `agent`, `admin`). The `super_admin` platform role, tenant provisioning, tenant status/suspension, and tenant-scoped data isolation have been removed rather than adapted.

---

## 1. Introduction

### 1.1 Purpose
This SRS restates the business requirements from the BRD as **testable functional and non-functional requirements** for engineering, reusing the FR numbering from `01-prd.md` where it still applies and adjusting requirements that assumed multi-tenancy.

### 1.2 Scope
Covers the full MVP system: an AI-first real estate CRM with a public site + customer portal and an admin portal, backed by a Python/FastAPI service layer, Supabase (Postgres), and Amazon Bedrock (Claude models). Single business, single deployment.

### 1.3 Definitions, Acronyms
| Term | Meaning |
|---|---|
| FR | Functional Requirement |
| NFR | Non-Functional Requirement |
| LLM | Large Language Model |
| NLU | Natural Language Understanding |

### 1.4 References
`00-project-overview.md`, `01-prd.md`.

---

## 2. Overall Description

### 2.1 Product Perspective
PropVista CRM is a standalone system with two front-end surfaces sharing one backend: the **public site + customer portal** and the **admin portal**. Both are React applications; the backend is a single FastAPI service that also handles AI orchestration against Amazon Bedrock.

### 2.2 User Classes and Characteristics
| Class | Technical proficiency | Primary need |
|---|---|---|
| Customer (visitor) | Low — general public | Fast, low-friction property discovery |
| Customer (registered) | Low | Persistence of saved items and requirement profile |
| Sales agent | Moderate | An efficient daily lead-working queue |
| Admin/Owner | Moderate | Control over listings, team, and AI behavior without engineering help |

### 2.3 Operating Environment
Web-based, mobile-responsive on the public/customer side; desktop-first on the admin side.

### 2.4 Design and Implementation Constraints
- Backend must be Python + FastAPI (async, for streaming chat and concurrent Bedrock calls).
- Data store must be Supabase Postgres, using Supabase Auth and Storage rather than custom-built equivalents.
- LLM calls must go through Amazon Bedrock using Anthropic Claude models.

### 2.5 Assumptions and Dependencies
System behavior depends on the availability and latency of Amazon Bedrock; degraded or unavailable AI service must not take down non-AI functionality (see NFR-Reliability).

---

## 3. Functional Requirements

### 3.1 Module 1 — AI Chatbot
- **FR1.1** The system shall display a chat widget on every public page, persisting across navigation within a session.
- **FR1.2** The system shall answer FAQs using admin-configured content.
- **FR1.3** The system shall retrieve live property data via backend tool calls rather than generating unverified answers.
- **FR1.4** The system shall capture visitor contact information mid-conversation and create a lead record.
- **FR1.5** The system shall allow the bot to offer a scheduled visit or callback and write the outcome to the CRM.
- **FR1.6** The system shall retain conversation history within a session, and across sessions for logged-in users.
- **FR1.7** The system shall escalate to a human agent when the bot cannot resolve a query or on explicit request.
- **FR1.8** The system shall log all conversations for admin review.

### 3.2 Module 2 — AI Search
- **FR2.1** The system shall accept free-text natural-language queries in a single search bar.
- **FR2.2** The system shall parse a query into structured constraints plus a semantic component.
- **FR2.3** The system shall rank results by combined structured + semantic relevance and display the match score to customers as a raw percentage.
- **FR2.4** The system shall provide auto-suggestions as the visitor types.
- **FR2.5** The system shall support voice input feeding the same query pipeline.
- **FR2.6** The system shall fall back to filter-based search if AI parsing fails or times out.

### 3.3 Module 3 — AI Recommendation (Requirement Analysis)
- **FR3.1** The system shall capture budget, location, property type, purpose, timeline, and must-have amenities via a multi-step form; a skipped step shall impose no constraint and shall renormalize remaining criterion weights.
- **FR3.2** The system shall return a ranked shortlist with a visible percentage match score and plain-language ✓/✗ reason lines.
- **FR3.3** The system shall let registered users save, edit, and delete a requirement profile.
- **FR3.4** The system shall notify a user of new matching properties, batched into one notification per recipient regardless of how many properties matched in a single event.
- **FR3.5** The system shall allow admin configuration of matching criteria weighting.
- **FR3.6** The system shall source all amenity values from one canonical amenity vocabulary; free-text amenity values shall not be permitted.

### 3.4 Module 4 — Property Listing (Browse/Search Results)
- **FR4.1** The system shall provide grid, list, and map view toggles.
- **FR4.2** The system shall support filters (price, type, bedrooms, location, amenities) combinable with AI search.
- **FR4.3** The system shall support sort by relevance, price, date added, and area.
- **FR4.4** The system shall allow favoriting without login (session-based) and migrate favorites to the account on registration/login.
- **FR4.5** The system shall paginate or infinite-scroll results.

### 3.5 Module 5 — Property Details
- **FR5.1** The system shall display image gallery, floor plan, amenities, price breakdown, location map, and nearby landmarks.
- **FR5.2** The system shall show "similar properties" using the recommendation engine.
- **FR5.3** The system shall keep a callback/visit CTA visible while scrolling.
- **FR5.4** The system shall display agent/builder contact info.
- **FR5.5** The system shall provide an EMI/affordability calculator.

### 3.6 Module 6 — Contact / Lead Capture
- **FR6.1** The system shall provide a contact form (modal and inline) with name, phone, email, message.
- **FR6.2** The system shall provide callback requests with a time-slot picker.
- **FR6.3** The system shall confirm submission to the user and optionally via SMS/email.
- **FR6.4** The system shall create exactly one lead per submission, tagged with source page and channel, and shall not create duplicates on retry (idempotency).

### 3.7 Module 7 — Customer Portal
- **FR7.1** The system shall show a dashboard of saved properties, requirement profile, inquiry history, and notifications.
- **FR7.2** The system shall provide notification channel preferences (email/SMS/in-app).
- **FR7.3** The system shall allow editing and deleting the saved requirement profile.

### 3.8 Module 8 — Admin Dashboard & Analytics
- **FR8.1** The system shall show KPI cards for total listings, active leads, conversion rate, and sessions (self-hosted session counter, labelled "Sessions", not third-party analytics).
- **FR8.2** The system shall chart lead source breakdown and property views over time.
- **FR8.3** The system shall show a recent activity feed.
- **FR8.4** The system shall prominently surface the unclaimed lead count.

### 3.9 Module 9 — Property Management
- **FR9.1** The system shall support CRUD for listings including photos, video, floor plans, and documents.
- **FR9.2** The system shall support bulk upload via CSV/Excel with validation and error reporting.
- **FR9.3** The system shall enforce a Draft → Pending Approval → Published workflow, configurable as to whether approval is required.
- **FR9.4** The system shall support status flags: Featured, Sold, On Hold, Archived.
- **FR9.5** The system shall automatically embed/index a listing for AI Search on save.

### 3.10 Module 10 — Lead & CRM Pipeline
- **FR10.1** The system shall enforce a fixed pipeline: New → Contacted → Site Visit Scheduled → Negotiation → Closed Won / Closed Lost.
- **FR10.2** The system shall create new leads unassigned in a shared "New" column visible to every agent, claimable by any agent; admins may reassign at any time.
  - **FR10.2a** Lead claim shall be atomic; simultaneous claim attempts shall resolve to exactly one owner.
  - **FR10.2b** The system shall support a mandatory stale-lead notification rule for unclaimed leads past a threshold.
- **FR10.3** The system shall support notes, call logs, and follow-up reminders per lead.
- **FR10.4** The system shall track lead source across all channels, including manually added walk-ins.
- **FR10.5** The system shall provide a table view sortable by unclaimed age, as an alternative to Kanban.

### 3.11 Module 11 — User & Role Management
- **FR11.1** The system shall support three fixed roles: `customer`, `agent`, `admin`.
- **FR11.2** The system shall support an invite flow for admin-portal users and disallow self-registration for those roles.
- **FR11.3** The system shall maintain an append-only audit log of admin actions, including denied actions, retained 7 years.

### 3.12 Module 12 — Agent/Broker Management
- **FR12.1** The system shall maintain agent profile, assigned listings, and assigned leads.
- **FR12.2** The system shall show a performance view: leads closed, response time, conversion rate.
- **FR12.3** The system shall show a leaderboard across agents.

### 3.13 Module 13 — AI Configuration
- **FR13.1** The system shall support chatbot configuration: greeting script, FAQ library, escalation rules.
- **FR13.2** The system shall provide a conversation log viewer with flagging.
- **FR13.3** The system shall support requirement-analysis weighting configuration.
- **FR13.4** The system shall surface basic AI Search performance visibility (e.g. no/low-result queries).

### 3.14 Module 14 — Content & Website Management (CMS)
- **FR14.1** The system shall allow editing homepage banners, blog posts, and static pages.
- **FR14.2** The system shall support SEO metadata fields per page/listing.

### 3.15 Module 15 — Reports & Notifications
- **FR15.1** The system shall generate reports (leads, sales, inventory) synchronously with a row cap, re-running the query on export rather than storing a saved report; the system shall reject an over-cap date range with guidance to narrow it, rather than silently truncating.
  - **FR15.1a** A sales report shall report `leads.deal_value`, not listing asking price.
- **FR15.2** The system shall support configurable notification rules (event → channel → recipients) across in-app, email, and SMS.
  - SMS shall be opt-in and capped.
  - Supported events: `new_lead`, `lead_escalated`, `lead_stale`, `lead_assigned`, `follow_up_due`, `property_pending_approval`, `listing_expiring`.
  - A notification dispatch failure shall never fail the action that triggered it.

### 3.16 Module 16 — Branding & Site Settings
- **FR16.1** The system shall support branding configuration (logo, colors, custom domain) with live preview and DNS verification before a custom domain is routed. *(Post-MVP — MVP ships one fixed palette; see §7.)*

---

## 4. Non-Functional Requirements

### 4.1 Performance
- **NFR-P1**: AI-assisted search, chat, and recommendation responses shall meet latency budgets defined per feature.
- **NFR-P2**: A newly published property shall become searchable and matchable within a defined, bounded sync window.
- **NFR-P3**: Report generation shall respect a row cap so that synchronous generation stays within an acceptable response time.

### 4.2 Scalability
- **NFR-S1**: Background work (embeddings, bulk import, notification dispatch) shall run asynchronously via a jobs mechanism rather than inline in the request path.

### 4.3 Security
- **NFR-SEC1**: A restricted-role user shall be unable to access or call APIs for features outside their permission set, enforced server-side.
- **NFR-SEC2**: The system shall log denied actions in the audit log, not only successful ones.
- **NFR-SEC3 (accepted risk)**: The admin portal operates without 2FA at MVP; this is a known, accepted risk, to be revisited via Supabase Auth MFA support.

### 4.4 Reliability & Availability
- **NFR-R1**: A notification/dispatch provider outage shall never block the action that triggered it (e.g. lead creation must succeed even if SendGrid/Twilio is down).
- **NFR-R2**: A failed background embedding job shall not be silent — it shall alert, and any published-but-unindexed property shall be surfaced to admins.

### 4.5 Usability
- **NFR-U1**: All public-site and customer-portal screens shall be usable on mobile viewports.
- **NFR-U2**: The admin portal shall be desktop-first; the Kanban pipeline view specifically is not required to be usable on a phone and shall fall back to a table view there.
- **NFR-U3**: Non-technical admin users shall be able to publish a CMS content change without developer involvement.
- **NFR-U4**: AI configuration changes shall take effect for new conversations/searches without requiring a deployment.

### 4.6 Maintainability
- **NFR-M1**: Conversion rate, active leads, and response time shall be computed in exactly one place (a single metrics service) and reused by the Dashboard, Agent leaderboard, and Reports modules.
- **NFR-M2**: Matching/weighting logic for AI Recommendation shall be configurable without code changes.

### 4.7 Auditability & Compliance
- **NFR-A1**: Admin actions and lead-stage changes shall be timestamped and logged for reporting and accountability.
- **NFR-A2**: Audit log entries shall be append-only (no update/delete).
- **NFR-A3**: SMS notifications sent to Indian phone numbers shall comply with DLT registration requirements before being enabled.

---

## 5. External Interface Requirements

### 5.1 User Interfaces
Two React front ends: the public/customer surface (mobile-responsive) and the admin portal (desktop-first).

### 5.2 API Interfaces
A single FastAPI backend exposes REST endpoints consumed by both front ends.

### 5.3 Software Interfaces
- Supabase (Postgres, Auth, Storage).
- Amazon Bedrock (Anthropic Claude models) for chat, search NLU, and recommendation reasoning.
- SendGrid (email) and Twilio (SMS) for notification dispatch.

---

## 6. Key Use Cases

| ID | Use case | Primary actor | Trigger | Outcome |
|---|---|---|---|---|
| UC-1 | Natural-language property search | Visitor | Free-text query in search bar | Ranked, scored results |
| UC-2 | Chatbot-assisted lead capture | Visitor | Ongoing chat conversation | Lead created with source = chatbot |
| UC-3 | Guided requirement matching | Visitor/registered customer | Completes multi-step wizard | Ranked shortlist with match reasons |
| UC-4 | Claim a lead | Agent | New lead appears in shared queue | Lead atomically assigned to one agent |
| UC-5 | Publish a property | Admin/Agent | Listing submitted for publish | Listing live, embedded/indexed for AI search |
| UC-6 | Bulk import listings | Admin | CSV/Excel upload | Rows validated; errors reported; valid rows created |
| UC-7 | Configure chatbot behavior | Admin | Edits greeting/FAQ/escalation rules | Changes live for next conversation, no deploy |
| UC-8 | Generate a sales report | Admin | Selects date range and filters | Report matches dashboard figures for same range |
| UC-9 | Stale lead alert | System (scheduled) | Lead unclaimed past threshold | Notification dispatched to configured recipients |

---

## 7. Post-MVP Notes

- Branding/theming (FR16.1) is fully specified but intentionally not built at MVP — the site ships on one fixed palette.
- Granular per-feature sub-roles (e.g. Marketing, Support) are deferred past MVP.

---

## 8. Traceability Summary

Every FR in §3 traces to its owning module in `01-prd.md` by shared numbering. Every NFR in §4 traces to a cross-cutting requirement in `01-prd.md` §18. Multi-tenant-specific requirements from the source docs (tenant isolation, tenant provisioning, tenant status, super admin) have been removed rather than mapped, per the project's single-business scope.

---

**Next document:** Customer Portal Functional Specification — ReactJS screens, workflows, and APIs for the public site and customer portal.
