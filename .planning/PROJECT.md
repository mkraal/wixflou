# Client Portal (Wix + Supabase + API Layer)

## What This Is

A private client portal built on the existing Wix website, where each authenticated client logs in and sees personalized content: their active projects synced from Caflou CRM, an order form with pre-filled details and auto-calculated pricing, and links to their Google Drive project backups. The portal is invite-only — the business admin creates accounts and links them to companies.

## Core Value

Each client can log in and immediately see only their own company's live project data — without the business team manually sending updates, links, or pricing by email.

## Requirements

### Validated

(None yet — ship to validate)

### Active

**Authentication & Access**
- [ ] Client can log in with email and password (Wix Members, invite-only)
- [ ] Client sees only their linked companies' data — no access to unlinked companies
- [ ] A member can be linked to multiple companies (sees projects from all linked companies)
- [ ] Admin can create a new client account and link it to one or more companies without developer involvement (mechanism TBD in Phase 0)

**Security**
- [ ] All data access from Wix goes through the API layer — no direct Supabase queries from Wix
- [ ] API validates Wix member session server-side before returning data
- [ ] Database credentials (service_role key) never leave the API layer
- [ ] API enforces client_id scoping on every query — defense in depth beyond application logic

**Monitoring**
- [ ] Consecutive sync failures alert the admin (email or Slack)
- [ ] API errors are logged with context for debugging
- [ ] External uptime check monitors the API and alerts on downtime

**Active Projects**
- [ ] Client can view all active projects for their company, synced from Caflou
- [ ] Project data is no more than 15 minutes stale
- [ ] Each project shows: name, status, assigned manager, last synced timestamp

**Orders**
- [ ] Client can submit an order with auto-filled name, company, and email (from Wix member profile)
- [ ] Order form shows total price calculated as: number of standard pages x client's rate
- [ ] Client rate is pulled from Caflou (not manually entered in portal)
- [ ] Order is created directly in Caflou via the API (Caflou is the source of truth)
- [ ] If Caflou is unreachable, client sees a clear error and can retry

**Project Backups**
- [ ] Each project shows a link to its Google Drive backup folder
- [ ] Drive links come from Caflou project attachments (no separate Drive integration)

### Out of Scope

- PM-level project filtering (each PM sees only their own projects) — v2, requires Caflou "assigned manager" API field
- Real-time sync via webhooks — 5-15 min polling is sufficient; webhooks can be added later
- Order status tracking in portal (orders are created in Caflou; syncing order status back is v2)
- Payment processing or invoicing
- File upload from client to business
- Chat or messaging
- Mobile app (responsive web only)
- Multi-language support
- Self-registration (invite-only only)

## Context

- **Existing CRM**: Caflou — source of truth for projects, tags, pricing, Drive links. Projects are tagged with company names to identify which client they belong to.
- **Existing website**: Live on Wix. Business user handles design/layout updates themselves via Wix editor. No developer needed for visual changes.
- **Caflou API**: REST API with Bearer token auth. Not yet tested — Phase 0 spike is required to validate tag filtering, custom attributes, response times, and pagination behavior.
- **Drive links**: Team does not yet consistently attach Google Drive links to Caflou projects. This workflow must be adopted before Phase 3 can deliver value.
- **Development approach**: AI-assisted — a developer works with an AI coding agent against the detailed plans. Estimates reflect this.
- **PRD reference**: Stakeholder brief in `docs/PRD-002-stakeholder-brief.md`.

## Architecture

```
Caflou (source of truth)
    |
    | External sync service (Make.com or Cloud Run Jobs, every 5-15 min)
    | Pulls: projects, pricing, Drive links — stamps clientId on each row
    v
Supabase / PostgreSQL (read cache — orders go directly to Caflou)
    ^
    | service_role key (never exposed outside API)
    |
API Layer (Supabase Edge Functions / Cloudflare Workers / TBD)
    | Validates Wix member session → resolves clientId → queries Supabase
    | Endpoints: GET /projects, GET /pricing, POST /orders (→ Caflou), GET /backups, POST /admin/link-member (if needed)
    ^
    | HTTPS calls from Wix backend (.jsw via wix-fetch)
    |
Wix Pages (pure presentation layer)
    ^
    | Wix Members (auth only — login, session, memberId)
```

**Wix's only jobs:** render pages, handle member login, forward requests to the API.
**API's only jobs:** validate sessions, enforce client-scoped queries, return filtered data, create orders in Caflou.
**Supabase's only jobs:** store synced data (read cache).
**The sync service's only job:** Caflou → Supabase pipeline.

### Sync Service — To Be Decided in Phase 0

The sync service runs every 5–15 minutes, pulls from Caflou, and writes to Supabase. No user-facing traffic.

| Option | Runtime | Free Tier | Notes |
|--------|---------|-----------|-------|
| Make.com | Visual no-code | 1,000 ops/mo | Drag-and-drop. Business user can inspect/modify. Built-in error notifications and retry. |
| Cloud Run Jobs (Google) | Any (containerized) | ~50K invocations/mo | Full code control, version-controlled, testable. Scheduled via Cloud Scheduler. Scales to zero. **Better fit for AI-assisted development.** |

**Trade-off:** Make.com is easier to hand off to a non-developer. Cloud Run Jobs gives full code control and is easier to maintain with AI agents (code is in a repo, not locked in a visual platform). Both are free at this scale.

### API Hosting — To Be Decided in Phase 0

| Option | Runtime | Free Tier | Notes |
|--------|---------|-----------|-------|
| Supabase Edge Functions | Deno/TS | 500K invocations/mo | Zero extra infra, deploys with DB. **Recommended starting point.** |
| Cloudflare Workers | JS/TS | 100K requests/day | Mature, fast. Good fallback if Edge Functions prove limiting. |
| Cloud Run (Google) | Any | 2M requests/mo | Full control, more setup. Consider if sync service also runs here. |
| Vercel / Netlify Functions | JS/TS | Varies | Easy DX, adds platform dependency. |

### Monitoring

Set up in Phase 1 alongside the first production deployment:

- **Sync health**: The sync service records success/failure status. Consecutive failures trigger an alert (email or Slack). Make.com has built-in error notifications; Cloud Run Jobs would write a status row to Supabase on each run and alert via a simple check (or Cloud Monitoring).
- **API error logging**: Supabase Edge Functions log to Supabase dashboard. Cloudflare Workers log to Workers dashboard. Either way, errors include endpoint, error type, and clientId for debugging.
- **Uptime monitoring**: External service (UptimeRobot or Better Stack, free tier) pings the API health endpoint on a schedule. Alerts admin if unreachable.

All monitoring uses free-tier tooling at this project's scale.

## Identity Mapping

`clientId` identifies a company. `memberId` identifies a person. One person can be linked to multiple companies.

**Tables:**
1. **`clients`** — one row per company: `clientId` (UUID), `caflouTag` (company name string used in Caflou)
2. **`client_members`** — join table: `memberId` (Wix member ID) ↔ `clientId` (UUID). One member can have multiple rows (multi-company access).
3. **`projects`** — every synced project row is stamped with `clientId` by the sync service (resolved via `caflouTag` match)

On every page load:
```
Wix page → .jsw backend → API (with Wix session token)
    API validates session with Wix → gets memberId
    API looks up clientIds from client_members WHERE member_id = ?
    API queries Supabase WHERE client_id IN (...) (using service_role key)
    API returns only that member's companies' data
```

`memberId` resolution and query scoping happen inside the API, not in Wix backend functions. The Wix `.jsw` layer only forwards the session — it never sees database credentials or constructs data queries.

**Admin onboarding:** Admin creates the Wix member, then links that member to their company(ies). The exact mechanism is a Phase 0 question:
- **Option A (preferred):** Member-to-company relationships are managed in Caflou (contacts linked to companies). The sync service populates `client_members` automatically. Admin only works in Caflou and Wix — no extra tooling.
- **Option B (fallback):** If Caflou doesn't expose contact-to-company relationships via API, a protected admin page in Wix calls `POST /admin/link-member` to manage `client_members` directly. Admin never touches Supabase.

**Phase 0 should confirm:** whether Caflou's company records expose a stable internal ID. If yes, use that as `clientId` directly — the sync service can derive it without a lookup. If not, generate a UUID in the `clients` table.

**Fragility point:** `caflouTag` is a company name string. If the company renames in Caflou, the `clients` table must be updated to match. This is the single point of coupling between Caflou and Supabase.

## Constraints

- **Platform**: Wix Velo — no npm packages in frontend; API called via `wix-fetch` in `.jsw` backend files
- **Wix limitations**: 14s backend timeout (mitigated by external sync), hourly scheduled jobs on standard plan (mitigated by external sync service)
- **Caflou API**: Untested — tag format, pagination, rate limits, and custom attribute shape all unknown until Phase 0
- **Drive links prerequisite**: Phase 3 is blocked until team adopts the Caflou attachment workflow
- **Data isolation**: Enforced in the API layer — Wix never queries Supabase directly. The API validates the Wix session, resolves the member's companies via `client_members`, and scopes all queries. Database credentials (service_role key) never leave the API.
- **Member-to-company mapping**: Managed in Supabase `client_members` table via a protected admin page in Wix. Admin never touches Supabase directly.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| API layer between Wix and Supabase | Without it: DB credentials in Wix, no safety net on query filtering, no audit trail. With it: credentials stay server-side, isolation enforced centrally, auditable. | — Pending (validate in Phase 0) |
| External sync service (not pure Wix) | Wix 14s timeout + hourly job limit makes in-Wix sync fragile. External service has no timeout, can run every 5 min. | — Pending |
| Sync service: Make.com vs Cloud Run Jobs | Make.com: visual, handoff-friendly. Cloud Run Jobs: code-controlled, AI-maintainable, version-controlled. Both free at this scale. | — Pending (decide in Phase 0) |
| Supabase instead of Wix Collections | Relational schema fits this domain better than NoSQL. SQL joins, proper tooling, not tied to Wix's data layer. | — Pending |
| Caflou as single source of truth | Projects, pricing, Drive links, and orders all originate in Caflou. Supabase is a read cache. Only `clients` originate outside Caflou. | — Pending |
| Company-level filtering for v1 | Simpler — tag maps to company, all PMs from same company see all company projects. PM-level is a permission layer on top, deferred to v2. | — Pending |
| Polling over webhooks | 5-15 min freshness is sufficient. Webhooks add endpoint management and reconciliation complexity. Add later if needed. | — Pending |
| Invite-only access | Admin controls who gets in. No self-registration. Reduces spam and ensures identity mapping is always set up before login. | — Pending |
| AI-assisted development | Developer + AI coding agent against detailed plans. Cuts scaffolding/boilerplate time significantly. Main bottleneck is Caflou API exploration and production testing, not code generation. | Active |

---
*Initialized: 2026-03-08*
