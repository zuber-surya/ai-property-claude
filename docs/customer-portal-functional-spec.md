# Customer Portal Functional Specification — PropVista CRM

> **Doc 3 of 9** in the new documentation set requested for this project.
> Built from `00-project-overview.md` and `01-prd.md` only.
>
> **Status:** Draft v1.0 · **Last updated:** July 2026
>
> **Note on APIs in this document:** endpoint paths below are proposed to keep this spec buildable; they are not yet ratified. The **API Specification** document (doc 7 of 9) is the source of truth once written — if it disagrees with a path here, the API Spec wins.

---

## 1. Purpose & Scope

This document specifies every screen in the **public site** (no login required) and **customer portal** (logged-in area) — both built in React — covering layout, user workflow, state handling, and the APIs each screen calls. It sits below the BRD/SRS (what must exist) and maps each screen back to the FR IDs from `01-prd.md` it fulfills.

---

## 2. Screen Inventory

| # | Screen | Auth required | Primary modules |
|---|---|---|---|
| 1 | Homepage | No | AI Search, CMS |
| 2 | Property Listing (search results) | No | AI Search, Module 4 |
| 3 | Property Details | No | Module 5 |
| 4 | AI Recommendation Wizard | No (save requires login) | Module 3 |
| 5 | Contact / Callback Modal | No | Module 6 |
| 6 | Login / Register | No | Auth |
| 7 | Customer Dashboard | Yes | Module 7 |
| 8 | Saved Properties | Yes | Module 7, 4 |
| 9 | Requirement Profile | Yes | Module 3, 7 |
| 10 | Inquiry History | Yes | Module 7, 10 |
| 11 | Notification Preferences | Yes | Module 7, 15 |
| 12 | Static / CMS Page (About, Careers, Terms, Blog) | No | Module 14 |
| — | AI Chatbot Widget (cross-page) | No (session-based); persists on login | Module 1 |

---

## 3. Cross-Cutting Components

### 3.1 AI Chatbot Widget
Persistent floating widget available on every public and customer-portal page.

- **Behavior:** Opens to a greeting from admin-configured settings (FR13.1). Streams responses token-by-token. Can call backend tools to look up live property data (FR1.3) rather than answer from memory.
- **State:** Conversation persists within the browser session via a `session_id`; for a logged-in user it also persists across sessions (FR1.6), keyed to `user_id`.
- **Lead capture:** If the bot collects contact info mid-conversation, it calls the lead-creation endpoint with `source = chatbot` (FR1.4).
- **Escalation:** A visible "Talk to a person" action or automatic escalation on repeated bot failure creates an `escalated` conversation flag an agent sees in near-real time (FR1.7).
- **APIs used:**
  - `POST /api/v1/chat/messages` — send a message, streamed response.
  - `GET /api/v1/chat/conversations/{id}` — resume a conversation.
  - `POST /api/v1/leads` — created internally by the bot's tool-call, not by the frontend directly.

### 3.2 Search Bar
Present on the homepage (hero) and as a persistent compact bar on the listing page.

- Accepts free text (FR2.1) and voice input (FR2.5, via the browser's speech-to-text feeding the same text field).
- Debounced auto-suggest as the user types (FR2.4).
- Falls back visibly to filter mode if AI parsing fails or times out (FR2.6) — the UI shows a non-blocking notice, not an error state.
- **APIs used:**
  - `GET /api/v1/search/suggest?q=` — autosuggest.
  - `POST /api/v1/search` — full query, returns structured + scored results.

### 3.3 Favorites (session → account migration)
A visitor can favorite a property without logging in (FR4.4), stored against a session ID. On login/registration, the frontend calls a migration endpoint so session favorites become account favorites.

- **APIs used:**
  - `POST /api/v1/favorites` / `DELETE /api/v1/favorites/{property_id}` — session or account scoped, resolved server-side by auth state.
  - `POST /api/v1/favorites/migrate` — called once, immediately after login/registration succeeds, if session favorites exist.

---

## 4. Screen Specifications

### 4.1 Homepage
**Purpose:** Entry point; drives visitors into AI search, the recommendation wizard, or featured listings.

**Key components:** Hero with search bar (§3.2); "Get matched" CTA into the Recommendation Wizard; featured/CMS-managed banners (FR14.1); featured property cards; static content sections (About teaser, testimonials) sourced from CMS.

**Workflow:** Visitor lands → either types a search query (routes to Property Listing with query pre-filled) or clicks "Get matched" (routes to Recommendation Wizard) or browses featured cards (routes to Property Details).

**APIs used:** `GET /api/v1/cms/homepage`, `GET /api/v1/properties/featured`.

**Related FRs:** FR2.1, FR2.4, FR3.1, FR14.1.

---

### 4.2 Property Listing (Search Results)
**Purpose:** Browse/filter/sort results from either an AI search query or manual filters.

**Key components:**
- View toggle: grid / list / map (FR4.1).
- Filter panel: price range, type, bedrooms, location, amenities (FR4.2) — amenities populated from the canonical vocabulary (FR3.6), not free text.
- Sort control: relevance, price asc/desc, date added, area (FR4.3).
- Result card: shows match score as a percentage when the result came from an AI query (FR2.3), a favorite toggle (FR4.4), and pagination or infinite scroll (FR4.5).

**Workflow:** Filters and AI search combine into a single query (both feed the same `POST /api/v1/search` request body) rather than conflicting — filters narrow, the AI query still contributes the semantic ranking component.

**States & edge cases:**
- Empty result set → explicit "no matches" state with a suggestion to broaden filters or try the Recommendation Wizard, never a blank grid.
- AI parse failure → silently degrade to filter-only search per FR2.6, with a small inline notice.
- Map view with zero geocoded results in the current viewport → show a "pan or zoom out" hint rather than an empty map.

**APIs used:** `POST /api/v1/search`, `GET /api/v1/properties` (filter-only fallback), `POST/DELETE /api/v1/favorites`.

**Related FRs:** FR2.1–FR2.6, FR4.1–FR4.5.

---

### 4.3 Property Details
**Purpose:** Full information on a single listing, and the primary lead-capture surface.

**Key components:**
- Image gallery, floor plan, amenities list (canonical vocabulary), price breakdown, location map, nearby landmarks (FR5.1).
- "Similar properties" carousel, reusing the recommendation engine seeded by this property's attributes (FR5.2).
- Sticky CTA bar (Request Callback / Schedule Visit) that stays visible while scrolling on both desktop and mobile without obstructing content (FR5.3).
- Agent/builder contact block (FR5.4).
- EMI/affordability calculator widget, client-side computation with admin-configurable interest-rate defaults (FR5.5).

**States & edge cases:** Missing floor plan, missing video, or partial media must not break the layout — each media block degrades gracefully to a placeholder or is omitted (FR5.1 acceptance criterion).

**APIs used:** `GET /api/v1/properties/{id}`, `GET /api/v1/properties/{id}/similar`, `POST /api/v1/leads` (from the sticky CTA), `GET /api/v1/properties/{id}/views` (fire-and-forget view-tracking beacon, FR8.2 dependency).

**Related FRs:** FR5.1–FR5.5, FR6.1–FR6.4.

---

### 4.4 AI Recommendation Wizard
**Purpose:** Guided, multi-step capture of buyer/renter needs, ending in a ranked, explainable shortlist.

**Key components:** Step-by-step form: budget range, location preference(s), property type, purpose (self-use/investment), timeline, must-have amenities (FR3.1). Each step is skippable; skipping removes that criterion from scoring rather than scoring it as zero, and the wizard's internal weighting renormalizes across the remaining answered criteria.

**Workflow:**
1. Visitor progresses through steps (skip allowed on every step except property type, which is required to scope results meaningfully — confirm in UI/UX spec).
2. On submission, the frontend calls the matching endpoint and renders a ranked shortlist with a percentage score and plain-language ✓/✗ reason lines per property (FR3.2), e.g. "Matches your budget and location; missing 1 amenity."
3. If logged in, the visitor can save the profile (FR3.3); if not, they're prompted to register to save it, with the in-progress answers carried through registration rather than lost.

**States & edge cases:** Sparse or zero exact matches must still return a ranked "closest matches" list with a clear explanation of what was relaxed — never an empty or broken result screen (FR3.1 acceptance criterion).

**APIs used:** `POST /api/v1/requirement-profiles` (create/save), `PUT /api/v1/requirement-profiles/{id}` (edit), `DELETE /api/v1/requirement-profiles/{id}`, `POST /api/v1/recommendations` (submit answers, get shortlist — usable without saving a profile).

**Related FRs:** FR3.1–FR3.6.

---

### 4.5 Contact / Callback Modal
**Purpose:** Lightweight lead capture reachable from multiple entry points (property details, listing cards, footer, chatbot handoff).

**Key components:** Name, phone, email, message fields (FR6.1); callback request variant adds a time-slot picker (FR6.2); on submit, a toast or inline confirmation, with optional SMS/email confirmation if that channel is enabled (FR6.3).

**Workflow:** Every submission — regardless of which page/modal triggered it — creates exactly one lead record tagged with source page and channel (FR6.4). The frontend must send a client-generated idempotency key so a duplicate submit (e.g. double-click, retry after timeout) does not create two lead records.

**APIs used:** `POST /api/v1/leads` (with `Idempotency-Key` header).

**Related FRs:** FR6.1–FR6.4.

---

### 4.6 Login / Register
**Purpose:** Account access for customers; unlocks saved searches, favorites, and the customer dashboard.

**Key components:** Standard email/password (Supabase Auth-backed); no special MFA requirement for customers.

**Workflow:** On successful registration/login, the frontend triggers the favorites migration call (§3.3) and, if the user arrived mid-Recommendation-Wizard, resumes and offers to save the in-progress profile.

**APIs used:** Supabase Auth SDK calls (not a custom backend endpoint); `POST /api/v1/favorites/migrate` post-login.

---

### 4.7 Customer Dashboard
**Purpose:** Logged-in home base — saved properties, requirement profile summary, inquiry history, notifications (FR7.1).

**Key components:** Summary cards linking into Saved Properties, Requirement Profile, Inquiry History, and a notification feed (in-app channel).

**States & edge cases:** All data strictly scoped to the logged-in user (FR7.1 acceptance criterion) — no cross-user leakage, enforced server-side, not just by hiding UI.

**APIs used:** `GET /api/v1/customer/dashboard` (aggregated summary), or individual calls to the endpoints in §4.8–§4.10 if the dashboard is composed client-side.

**Related FRs:** FR7.1.

---

### 4.8 Saved Properties
**Purpose:** List of favorited properties, editable from the same card pattern used on the Listing screen.

**APIs used:** `GET /api/v1/favorites`, `DELETE /api/v1/favorites/{property_id}`.

**Related FRs:** FR4.4, FR7.1.

---

### 4.9 Requirement Profile
**Purpose:** View, edit, or delete the saved profile from the Recommendation Wizard (FR7.3); re-running it against the current inventory refreshes the shortlist.

**Workflow:** Editing re-opens the wizard pre-filled with saved answers. Deleting removes the profile and stops match notifications for it (FR3.4 dependency).

**APIs used:** `GET /api/v1/requirement-profiles/{id}`, `PUT /api/v1/requirement-profiles/{id}`, `DELETE /api/v1/requirement-profiles/{id}`.

**Related FRs:** FR3.3, FR3.4, FR7.3.

---

### 4.10 Inquiry History
**Purpose:** Every lead/inquiry the customer has submitted (contact forms, callback requests, chatbot-originated), with status where visible (e.g. "callback scheduled").

**APIs used:** `GET /api/v1/customer/inquiries`.

**Related FRs:** FR7.1.

---

### 4.11 Notification Preferences
**Purpose:** Let the customer choose channels (email/SMS/in-app) for new-match notifications (FR7.2).

**Workflow:** Preference change takes effect on the next batched notification event (see FR3.4 batching behavior — one notification per event, not one per matched property).

**APIs used:** `GET /api/v1/customer/notification-preferences`, `PUT /api/v1/customer/notification-preferences`.

**Related FRs:** FR3.4, FR7.2.

---

### 4.12 Static / CMS Page
**Purpose:** Renders admin-authored content — About, Careers, Terms, blog posts — published from the admin CMS (Module 14).

**Key components:** Generic content renderer keyed by page slug; SEO metadata (title, meta description) rendered into the page head from CMS fields (FR14.2).

**APIs used:** `GET /api/v1/cms/pages/{slug}`, `GET /api/v1/cms/blog`.

**Related FRs:** FR14.1, FR14.2.

> **Open dependency:** the project overview (`00-project-overview.md` §9.1) notes there is currently no public CMS read endpoint specified — this screen's APIs above are proposed pending that gap being closed in the API Specification document.

---

## 5. Responsive / Mobile Requirements

All screens in this document must be usable on mobile viewports (cross-cutting NFR from the SRS). The chatbot widget, search bar, and sticky CTA in particular need explicit mobile layouts since they persist across scroll and must not overlap each other — sequencing/positioning to be finalized in the UI/UX Specification.

---

## 6. Open Questions

- Which Recommendation Wizard steps (if any) are mandatory vs. skippable — property type is assumed required above; needs product confirmation.
- Exact shape of the aggregated dashboard endpoint vs. client-composed calls (§4.7) — an implementation choice for the Backend Technical Design.
- Public CMS read endpoint is not yet specified (see §4.12 note) — blocks Module 14 on the customer side until resolved.

---

**Next document:** Admin Portal Functional Specification — ReactJS screens, workflows, and APIs for the admin/agent/super-admin surface.
