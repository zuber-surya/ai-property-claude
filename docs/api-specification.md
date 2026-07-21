# API Specification — PropVista CRM

> **Doc 7 of 9** in the new documentation set requested for this project.
> Built from `00-project-overview.md`, `01-prd.md`, and consistent with docs 3–6 (single-tenant, three roles: `customer`, `agent`, `admin`).
>
> **Status:** Draft v1.0 · **Last updated:** July 2026
>
> This document is the **source of truth** for endpoint contracts. Where earlier docs (3–5) proposed a path pending this spec, this document ratifies or corrects it. It also closes the three endpoint gaps flagged in `00-project-overview.md` §9.1: the public CMS read endpoint, requirement-profile delete, autosuggest, and resend/revoke invite.

---

## 1. Conventions

- **Base URL:** `/api/v1`
- **Auth:** Bearer JWT (Supabase Auth) on every authenticated route. Public routes (search, listings, chat, contact) require no auth; session state is carried via an `X-Session-Id` header.
- **Roles enforced server-side:** `customer`, `agent`, `admin`. Admin routes require `agent` or `admin` unless noted.
- **Pagination:** cursor-based — `?cursor=` / `?limit=` (default 20, max 100), response includes `next_cursor`.
- **Idempotency:** endpoints that create a record from a user-initiated action (leads, bulk uploads) accept an `Idempotency-Key` header.
- **Errors:** uniform envelope —
  ```json
  { "error": { "code": "string", "message": "string", "details": {} } }
  ```
  Standard HTTP status codes apply (400 validation, 401 unauthenticated, 403 forbidden, 404 not found, 409 conflict, 422 unprocessable, 429 rate-limited, 500 server error).
- **Timestamps:** ISO 8601, UTC.
- **Money:** numeric strings (never floats) to avoid precision loss, e.g. `"price": "8500000.00"`.

---

## 2. Public API

### 2.1 Search
| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/search` | none | Full AI + filter search. Body: `{ query?, filters?, sort?, cursor? }`. Returns scored results (FR2.1–FR2.3) or filter-only results if AI parsing fails/times out (FR2.6). |
| GET | `/search/suggest?q=` | none | Autosuggest as the user types (FR2.4). Closes gap **G3** from `00-project-overview.md` §9.1. |

**`POST /search` response (excerpt):**
```json
{
  "results": [
    {
      "property_id": "uuid",
      "title": "3BHK in Tech Park Road",
      "price": "8500000.00",
      "match_score": 92,
      "match_reasons": ["Matches your budget", "Matches your location", "Missing 1 amenity: gym"]
    }
  ],
  "fallback_mode": false,
  "next_cursor": "opaque-string"
}
```

### 2.2 Properties
| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/properties` | none | Filter/sort/paginate published listings (FR4.1–FR4.5). |
| GET | `/properties/featured` | none | Homepage featured set. |
| GET | `/properties/{id}` | none | Full detail (FR5.1). |
| GET | `/properties/{id}/similar` | none | Recommendation-engine-backed similar properties (FR5.2). |
| POST | `/properties/{id}/views` | none | Fire-and-forget view beacon (FR8.2 dependency). Body: `{ session_id }`. |

### 2.3 Chat
| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/chat/messages` | none (session or user) | Send a message; server-sent-events streamed response. Body: `{ conversation_id?, message, session_id }` (FR1.1–FR1.7). |
| GET | `/chat/conversations/{id}` | session/user match | Resume a conversation (FR1.6). |

### 2.4 Leads
| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/leads` | none (session or user) | Create a lead from any capture channel. Requires `Idempotency-Key`. Body: `{ source, property_id?, contact_name, contact_phone, contact_email, message?, requested_callback_at? }` (FR6.1–FR6.4). |

### 2.5 Favorites
| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/favorites` | session or user | List favorites for the current session/account. |
| POST | `/favorites` | session or user | Add. Body: `{ property_id }` (FR4.4). |
| DELETE | `/favorites/{property_id}` | session or user | Remove. |
| POST | `/favorites/migrate` | user (post-login) | Move session favorites onto the account. |

### 2.6 Requirement Profiles / Recommendation
| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/recommendations` | none | Submit wizard answers, get a ranked shortlist without saving a profile (FR3.1, FR3.2). |
| POST | `/requirement-profiles` | user | Save a profile (FR3.3). |
| GET | `/requirement-profiles/{id}` | owner | Read. |
| PUT | `/requirement-profiles/{id}` | owner | Edit. |
| DELETE | `/requirement-profiles/{id}` | owner | Soft-delete. Closes gap **G8** — the column existed, the endpoint didn't. |

### 2.7 Customer Account
| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/customer/dashboard` | user | Aggregated summary for FR7.1. |
| GET | `/customer/inquiries` | user | Inquiry/lead history. |
| GET | `/customer/notification-preferences` | user | FR7.2. |
| PUT | `/customer/notification-preferences` | user | FR7.2. |

### 2.8 CMS (public read)
| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/cms/homepage` | none | Homepage banners + sections. |
| GET | `/cms/pages/{slug}` | none | Static page by slug (FR14.1). |
| GET | `/cms/blog` | none | Blog listing. |
| GET | `/cms/blog/{slug}` | none | Blog post detail. |

Closes gap **G5** from `00-project-overview.md` §9.1 — a public CMS read path did not previously exist even though admins could publish content.

---

## 3. Admin API

All routes below require role `agent` or `admin` unless marked **admin only**.

### 3.1 Dashboard
| Method | Path | Description |
|---|---|---|
| GET | `/admin/dashboard/kpis` | FR8.1, backed by `metrics_service`. |
| GET | `/admin/dashboard/activity` | FR8.3. |

### 3.2 Properties
| Method | Path | Description |
|---|---|---|
| GET | `/admin/properties` | List with status filter. |
| POST | `/admin/properties` | Create (FR9.1). |
| GET | `/admin/properties/{id}` | Detail. |
| PUT | `/admin/properties/{id}` | Update. |
| DELETE | `/admin/properties/{id}` | **admin only.** |
| POST | `/admin/properties/{id}/approve` | **admin only** (FR9.3). |
| POST | `/admin/properties/{id}/reject` | **admin only.** Body: `{ reason }`. |
| POST | `/admin/properties/bulk-status` | Batch status/flag update (FR9.4). |
| POST | `/admin/properties/bulk-upload` | Start a bulk import job (FR9.2). |
| GET | `/admin/properties/bulk-upload/{job_id}` | Row-level progress/results. |

### 3.3 Leads
| Method | Path | Description |
|---|---|---|
| GET | `/admin/leads?view=kanban\|table` | FR10.1, FR10.5. |
| GET | `/admin/leads/{id}` | Full record. |
| POST | `/admin/leads/{id}/claim` | Atomic claim (FR10.2a) — 409 if already claimed. |
| POST | `/admin/leads/{id}/stage` | Move stage. Body: `{ to_stage }`. |
| POST | `/admin/leads/{id}/notes` | Note/call log/follow-up (FR10.3). |
| POST | `/admin/leads/{id}/follow-up` | Set a reminder. |

**`POST /admin/leads/{id}/claim` response on conflict:**
```json
{ "error": { "code": "already_claimed", "message": "This lead was claimed by another agent.", "details": { "owner_id": "uuid" } } }
```
Status `409`.

### 3.4 Users & Roles (admin only)
| Method | Path | Description |
|---|---|---|
| GET | `/admin/users` | List. |
| POST | `/admin/users/invite` | FR11.2. |
| POST | `/admin/users/{id}/resend-invite` | Closes an open gap from `00-project-overview.md` §9.1. |
| POST | `/admin/users/{id}/revoke-invite` | Same. |
| PUT | `/admin/users/{id}/role` | Restricted to `customer`/`agent`/`admin` — no other value accepted. |

### 3.5 Audit Log (admin only)
| Method | Path | Description |
|---|---|---|
| GET | `/admin/audit-log` | FR11.3, filterable, read-only. |

### 3.6 Agents
| Method | Path | Description |
|---|---|---|
| GET | `/admin/agents` | FR12.1. |
| GET | `/admin/agents/{id}` | Profile + workload. |
| GET | `/admin/agents/leaderboard` | FR12.2, FR12.3, backed by `metrics_service`. |

### 3.7 AI Configuration (admin only)
| Method | Path | Description |
|---|---|---|
| GET | `/admin/ai-config/chatbot` | FR13.1. |
| PUT | `/admin/ai-config/chatbot` | Takes effect on next conversation, no deploy (NFR-U4). |
| GET | `/admin/ai-config/search-insights` | FR13.4. |
| GET | `/admin/ai-config/recommendation-weights` | FR13.3, FR3.5. |
| PUT | `/admin/ai-config/recommendation-weights` | Same. |
| POST | `/admin/ai-config/recommendation-weights/preview` | Runs a sample profile against proposed weights before saving. Closes the previously-open "weights preview" gap. |

### 3.8 Conversation Log (admin only)
| Method | Path | Description |
|---|---|---|
| GET | `/admin/chat/conversations` | FR13.2, scoped per `01-prd.md` §18a (agent sees own leads' transcripts; colleagues' leads read-only). |
| POST | `/admin/chat/conversations/{id}/flag` | Toggle review flag. |

### 3.9 CMS (admin write)
| Method | Path | Description |
|---|---|---|
| GET | `/admin/cms/pages` | List, incl. drafts. |
| POST | `/admin/cms/pages` | FR14.1, FR14.2. |
| PUT | `/admin/cms/pages/{id}` | |
| GET | `/admin/cms/banners` | |
| POST | `/admin/cms/banners` | |
| PUT | `/admin/cms/banners/{id}` | |

### 3.10 Reports (admin only)
| Method | Path | Description |
|---|---|---|
| POST | `/admin/reports/leads` | Body: filters + date range. `?export=pdf\|xlsx` variant. Row-capped, synchronous (FR15.1). |
| POST | `/admin/reports/sales` | Reads `leads.deal_value` (FR15.1a). |
| POST | `/admin/reports/inventory` | |

**Row-cap error:**
```json
{ "error": { "code": "date_range_too_large", "message": "Narrow the date range and try again.", "details": { "max_rows": 50000 } } }
```
Status `422` — never a silently truncated 200.

### 3.11 Notification Rules (admin only)
| Method | Path | Description |
|---|---|---|
| GET | `/admin/notification-rules` | FR15.2. |
| POST | `/admin/notification-rules` | |
| PUT | `/admin/notification-rules/{id}` | The `lead_stale` rule (`is_mandatory=true`) rejects a request that would fully disable it — only the threshold is editable (FR10.2b). |

### 3.12 Branding (admin only, post-MVP)
| Method | Path | Description |
|---|---|---|
| GET | `/admin/branding` | FR16.1. |
| PUT | `/admin/branding` | Specced, not active pre-MVP. DNS verification required before a custom domain field is accepted as routable. |

---

## 4. Common Schemas

**Property (public view):**
```json
{
  "id": "uuid",
  "title": "string",
  "status": "published",
  "flag": "featured",
  "property_type": "apartment",
  "listing_type": "sale",
  "price": "8500000.00",
  "bedrooms": 3,
  "bathrooms": 2,
  "area_sqft": 1450,
  "location_text": "string",
  "amenities": ["gymnasium", "swimming_pool"],
  "media": [{ "type": "photo", "url": "string" }]
}
```

**Lead:**
```json
{
  "id": "uuid",
  "source": "chatbot",
  "stage": "new",
  "owner_id": null,
  "contact_name": "string",
  "contact_phone": "string",
  "contact_email": "string",
  "deal_value": null,
  "created_at": "2026-07-17T10:00:00Z"
}
```

**User (admin-visible):**
```json
{
  "id": "uuid",
  "email": "string",
  "role": "agent",
  "full_name": "string"
}
```

---

## 5. Error Code Reference

| Code | Status | Meaning |
|---|---|---|
| `validation_error` | 400 | Request body failed schema validation |
| `unauthenticated` | 401 | Missing/invalid auth token |
| `forbidden` | 403 | Authenticated but role doesn't permit this action |
| `not_found` | 404 | Entity doesn't exist or isn't visible to the caller |
| `already_claimed` | 409 | Lead claim lost the race (FR10.2a) |
| `duplicate_submission` | 409 | Idempotency key already used |
| `date_range_too_large` | 422 | Report request exceeds the row cap (FR15.1) |
| `rate_limited` | 429 | SMS or other capped-channel limit hit (FR15.2) |

---

## 6. Open Questions

- Exact `metrics_service` response shape shared across `/admin/dashboard/kpis`, `/admin/agents/leaderboard`, and `/admin/reports/*` — should be a single reusable schema; needs a follow-up pass once the Backend Technical Design's service interfaces are implemented.
- Whether `/chat/messages` streaming uses SSE or a WebSocket — SSE assumed above for simplicity with FastAPI's async support; revisit if bidirectional push (e.g. live agent takeover) is needed later.
- Rate-limit thresholds (`rate_limited`) are not yet numerically defined.

---

**Next document:** UI/UX Specification — screen navigation, user journeys, and component behavior.
