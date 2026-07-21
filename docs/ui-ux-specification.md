# UI/UX Specification — PropVista CRM

> **Doc 8 of 9** in the new documentation set requested for this project.
> Built from `00-project-overview.md`, `01-prd.md`, and consistent with docs 3–4 (Customer/Admin Functional Specs) and doc 7 (API Specification).
>
> **Status:** Draft v1.0 · **Last updated:** July 2026

---

## 1. Purpose & Scope

This document specifies **navigation structure, user journeys, and component-level behavior** across both React front ends. It sits above the per-screen functional specs (docs 3–4), which define what each screen contains; this document defines how a person moves between screens and how shared components behave everywhere they appear.

---

## 2. Design Principles

- **One fixed visual palette at MVP.** Branding configuration exists in the schema and API but isn't switched on for MVP (FR16.1) — every screen ships with the same look.
- **A reserved AI-signal color.** Whatever palette ships, one color family is reserved specifically to mark AI-generated content (match scores, AI search results, chatbot messages) so a user can distinguish "the system inferred this" from "this is raw data" at a glance. This color is not available for future branding overrides even post-MVP.
- **Mobile-first on the customer side, desktop-first on the admin side.** These are genuinely different design targets, not one responsive layout stretched two ways — the admin Kanban board in particular is designed for a mouse and a wide viewport and explicitly falls back to a table on narrow ones (NFR-U2).
- **AI features degrade visibly, not silently.** Every AI-dependent surface (search, chat, recommendation) has a defined fallback or error state a user can see and understand — never a blank screen or a spinner that never resolves.

---

## 3. Navigation Structure

### 3.1 Public site + customer portal (sitemap)

```
/                              Homepage
/search                        Property Listing (search results)
/properties/{id}                Property Details
/get-matched                    AI Recommendation Wizard
/login, /register                Auth
/account                        Customer Dashboard        (auth required)
  /account/saved                 Saved Properties
  /account/requirement-profile    Requirement Profile
  /account/inquiries              Inquiry History
  /account/notifications          Notification Preferences
/about, /careers, /terms, /blog  CMS static/blog pages
```

The chatbot widget and search bar are not routes — they're persistent components layered over every page in this tree (see §5).

### 3.2 Admin portal (sitemap)

```
/admin                          Dashboard
/admin/properties                Property List
  /admin/properties/{id}          Property Detail/Edit
  /admin/properties/bulk-upload    Bulk Upload
  /admin/properties/approvals      Approval Queue
/admin/leads                     Lead Pipeline (Kanban, default)
  /admin/leads?view=table          Lead Pipeline (Table)
  /admin/leads/{id}                 Lead Detail
/admin/users                     Users & Roles              (admin only)
/admin/audit-log                  Audit Log                  (admin only)
/admin/agents                    Agents
  /admin/agents/leaderboard         Leaderboard
/admin/ai-config
  /admin/ai-config/chatbot          Chatbot config           (admin only)
  /admin/ai-config/search            Search insights          (admin only)
  /admin/ai-config/recommendation    Weighting config         (admin only)
  /admin/ai-config/conversations     Conversation log viewer  (admin only)
/admin/cms                        CMS Pages & Banners        (admin only)
/admin/reports                    Reports                    (admin only)
/admin/notification-rules          Notification Rules         (admin only)
/admin/branding                   Branding Settings          (admin only, post-MVP)
```

An `agent`-role login lands on `/admin` and sees Dashboard, Properties (read + draft), Leads, and Agents/Leaderboard in the nav; everything marked admin-only is hidden from the nav entirely for an agent, not just disabled (though the API also enforces this server-side per NFR-SEC1).

---

## 4. User Journeys

### 4.1 Visitor discovery → lead (public site)

1. Lands on Homepage → types a query into the search bar **or** clicks "Get matched."
2. **Search path:** routes to `/search` with the query pre-filled; can refine with filters; opens a result into Property Details.
3. **Wizard path:** completes the multi-step form at `/get-matched`; sees a ranked shortlist with reasons; opens a result into Property Details.
4. On Property Details, uses the sticky CTA to request a callback or schedule a visit → Contact/Callback Modal → confirmation.
5. At any point, can instead open the chatbot widget and reach the same outcome (contact captured mid-conversation) without leaving the current page.

**Journey principle:** all three entry points (search, wizard, chat) are equally valid front doors into the same underlying property inventory and the same lead-capture outcome — no path is a "lesser" fallback of another.

### 4.2 Registered customer → saved profile revisit

1. Registers (from the wizard's save prompt, or directly via `/register`).
2. Session favorites and any in-progress wizard answers migrate onto the new account automatically — the customer should not have to redo anything they'd already done as an anonymous visitor.
3. Later returns, logs in, lands on `/account`, opens Requirement Profile, edits it, and re-triggers matching against current inventory.
4. Receives a batched notification (per FR3.4) when new matching properties are published, and can jump directly from that notification to the updated shortlist.

### 4.3 Agent → lead-to-close (admin portal)

1. Logs in, lands on `/admin`, sees Unclaimed Lead Count prominently on the Dashboard.
2. Opens `/admin/leads`, sees the shared New column, claims a lead by dragging it into Contacted.
3. Works the lead from Lead Detail — logs a call, sets a follow-up reminder, moves it through stages as the deal progresses.
4. Sees their own performance reflected on the Leaderboard as the lead closes.

**Journey principle:** an agent should never need to leave the Kanban/Table view and the Lead Detail screen to complete a normal day of work — those two screens are the primary workspace.

### 4.4 Admin → listing publish → AI-searchable

1. Creates or bulk-uploads listings at `/admin/properties`.
2. If approval is required, the listing sits in the Approval Queue until reviewed.
3. On publish, the admin sees an immediate save confirmation; the listing becomes AI-searchable shortly after, once the background embedding job completes — the UI does not block the admin waiting for this.
4. If embedding fails, this must surface as a visible flag on the property (not a silent gap) so the admin knows to investigate rather than wondering why a live listing never appears in search results.

### 4.5 Admin → AI configuration feedback loop

1. Reviews AI Search insights (`/admin/ai-config/search`) to see what buyers are searching for that isn't matching inventory.
2. Adjusts chatbot FAQ content or recommendation weighting accordingly.
3. Changes apply to the next conversation/search immediately — the admin can verify by opening a fresh chat or search on the live site right after saving, with no deployment delay.

---

## 5. Cross-Cutting Component Behavior

### 5.1 AI Chatbot Widget
- **Position:** fixed bottom-right on desktop, collapses to a bottom bar on mobile; never overlaps the sticky CTA on Property Details (§5.3) — one is bottom-right, the other spans the bottom edge, with the chatbot's collapsed state sitting above it.
- **States:** collapsed (default) → expanded (conversation) → escalated (visibly different header treatment, e.g. "Connecting you to an agent") → closed.
- **Streaming:** assistant messages render token-by-token, not as a single block after a delay, so the user has immediate feedback that the system is responding.
- **Escalation state persists** across widget collapse/expand — a user who escalates and then browses to another page should return to the escalated state, not a fresh greeting.

### 5.2 Search Bar
- **Debounce:** 300ms on autosuggest keystrokes.
- **Fallback notice:** if AI parsing fails/times out, a small non-blocking inline banner reads roughly "Showing filter-based results" — never a hard error, since the user's search still functionally succeeds via the fallback.
- **Match score display:** shown only on AI-parsed results, as a badge using the reserved AI-signal color (§2), e.g. "92% match" — never shown on plain filter results, since a filter match isn't a model-generated score.

### 5.3 Sticky CTA (Property Details)
- Pins to the bottom of the viewport on scroll on both desktop and mobile.
- On mobile, collapses to two buttons (Callback / Visit) without labels beyond icon + short text, to avoid consuming excessive vertical space on small screens.
- Never obstructs the last visible content element — maintains a bottom safe-area padding on the page content equal to its own height.

### 5.4 Result / Listing Card
- Consistent card used across Homepage (featured), Listing page (grid/list), Saved Properties, and "Similar properties."
- Favorite toggle is always in the same corner position across every context it appears in.
- Match score badge (when present) appears in the same position as the price, distinguishing itself by the reserved AI-signal color rather than layout — so the card layout doesn't shift between AI and non-AI contexts.

### 5.5 Lead Kanban Card
- Shows: contact name, source icon, time-in-stage (not just time-since-created — this is what makes a stalled lead visually obvious without opening it).
- Unclaimed cards in the New column carry a distinct visual treatment (not just column position) so they read as "needs action" even in a screenshot or when scrolled.
- Drag interaction provides a clear drop-target highlight; a failed claim (409 conflict, per the API spec) animates the card back to its origin with an inline message, not a silent no-op.

### 5.6 Multi-Step Wizard (Recommendation)
- Persistent step indicator (e.g. "Step 3 of 6") with the ability to jump backward to any completed step, but not forward past the current one.
- Every step has an explicit "Skip this" affordance distinct from leaving a field blank and clicking next — skip is a deliberate action that maps to "no constraint," not an accidentally-empty required field (FR3.1).
- Submission shows a loading state that explains what's happening ("Matching you with properties…") rather than a generic spinner, since this call can take longer than a typical page load due to AI processing.

### 5.7 Notification Toast / In-App Feed
- Toasts confirm transient actions (form submitted, favorite added); the in-app notification feed (Customer Dashboard, Admin activity feed) persists until dismissed or read.
- A batched notification (FR3.4) renders as one item ("12 new properties match your saved search") — never as multiple stacked toasts for a single triggering event.

---

## 6. Responsive Breakpoints

| Breakpoint | Target | Applies to |
|---|---|---|
| `< 640px` | Mobile | Public site, customer portal — full functionality required (NFR-U1) |
| `640–1024px` | Tablet | Public site, customer portal — treated as mobile layout with more breathing room, not a distinct third layout |
| `> 1024px` | Desktop | Admin portal baseline; public site gets its widest layout here |

The admin portal below `1024px` is not a supported target except for the Lead Pipeline, which has an explicit Table-view fallback (§3.5 in the Admin Portal Functional Spec) — every other admin screen may be usable but is not designed or tested for narrower viewports at MVP.

---

## 7. Global State Patterns

- **Empty states** are written specifically per screen, never a generic "No data" — e.g. Property Listing's empty state suggests broadening filters or trying the Recommendation Wizard (FR3.1 acceptance criterion), not just "0 results."
- **Loading states** for AI-backed actions (search, chat, recommendation) describe what's happening rather than showing a bare spinner, since these can take longer than a typical request.
- **Error states** distinguish recoverable failures (AI fallback, retry-able network error) from ones that need the user to change something (validation error) — the former offers a retry or silently degrades, the latter points at the specific field or input.

---

## 8. Accessibility Notes

- All interactive components (chatbot widget, wizard, Kanban drag-and-drop) need a non-pointer-dependent path: the Kanban claim action, in particular, needs a keyboard/menu-based "Claim" alternative to drag-and-drop, both for accessibility and as the natural fallback on the Table view.
- The reserved AI-signal color (§2) must not be the only indicator of AI-generated content — pair it with a label or icon so the distinction doesn't depend on color perception alone.
- Streaming chat responses should be announced incrementally to screen readers in a way that doesn't re-read the entire message on every token — an implementation detail for the frontend build, flagged here so it isn't missed.

---

## 9. Open Questions

- Exact reserved AI-signal color value — a visual design decision outside this document's scope, but the constraint (one color, platform-owned, never override-able) is fixed here.
- Whether the Recommendation Wizard's step order is fixed or partially reorderable based on the visitor's first answer (e.g. "investment" purpose might prioritize different early questions than "self-use") — a product decision, not yet made.
- Mobile Kanban alternative beyond Table view (e.g. a swipe-based single-column mobile Kanban) — currently out of scope per NFR-U2, revisit only if user research shows agents need mobile pipeline access.

---

**Next document:** Development Operating System (DevOS) — phases, tasks, Claude Code prompts, progress tracking, and testing strategy.
