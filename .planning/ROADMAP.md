# Roadmap: Client Portal (Wix + Supabase + API Layer)

**4 phases** | **33 requirements mapped** | All v1 requirements covered ✓

Estimates assume AI-assisted development — a developer working with an AI coding agent against these detailed plans. The AI handles scaffolding, boilerplate, and repetitive code. The developer focuses on integration, testing, and decisions that require human judgment.

---

## Phase 0: Spike — Validate & Prove the Stack

**Goal:** Confirm the entire technical approach works before writing production code. Answer every open question that affects scope and architecture. Prove the secure data flow end-to-end: Caflou → Supabase → API → Wix.

**Requirements:** AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05, AUTH-06, SEC-01, SEC-02, SEC-03, SEC-04, SYNC-01, SYNC-02, SYNC-03, SYNC-04, SYNC-05, SYNC-06

**Estimate:** 11–17 h

**What gets built:**
- Wix dev site with Velo enabled and a test member login
- Supabase project with schema: `clients`, `client_members`, `projects`, `client_pricing`
- API layer proof-of-concept (Supabase Edge Functions or Cloudflare Workers): session validation + client-scoped query endpoints
- Sync proof-of-concept (Make.com scenario or Cloud Run Job) that pulls projects from Caflou and writes them to Supabase
- Wix backend `.jsw` functions that call the API (not Supabase directly)
- A test page that shows logged-in member's projects fetched via the Wix → API → Supabase chain

**Questions this phase must answer:**
1. Are project tags accessible on the project object via Caflou API? (Tags are how we match projects to companies — must be readable. Dataset is small (~50-100 customers), so no server-side filtering needed — sync fetches all and matches locally.)
2. Are custom attributes (`custom_column_gdrive_link`, `custom_column_price_per_page`) accessible via API? (Pricing and backup links depend on these.)
3. What Caflou object should an order create — a task, a project, or something else? (Depends on how the business currently tracks orders in Caflou.)
4. Does Caflou expose contact-to-company relationships via API? (If yes: member-to-company mapping syncs automatically — preferred. If no: build a simple admin page in Wix as fallback.)
5. Which sync service fits best — Make.com (visual, handoff-friendly) or Cloud Run Jobs (code-controlled, AI-maintainable)?
6. Which API host works best — Supabase Edge Functions, Cloudflare Workers, or another option?
7. Can the API reliably validate a Wix member session and resolve their clientId?
8. Does the full secure chain work end-to-end: Wix login → API session validation → clientId resolution → scoped Supabase query → data returned to Wix page?

**Success criteria:**
1. A logged-in Wix test member sees only projects belonging to their linked companies (no access to unlinked companies)
2. Data flows through the API layer — Wix never queries Supabase directly
3. API validates the Wix session server-side before returning any data
4. A project update in Caflou appears in the test portal within 15 minutes
5. All open questions above are documented with confirmed answers
6. Sync service and API host are chosen and documented
7. No hard blockers found — or blockers identified with mitigation plan

---

## Phase 1: Active Projects

**Goal:** Clients can log in and see their live project list. This is the core of the portal. Includes basic monitoring so problems are caught before clients notice.

**Requirements:** PROJ-01, PROJ-02, PROJ-03, PROJ-04, MON-01, MON-02, MON-03

**Estimate:** 14–24 h

**What gets built:**
- Production Wix member area (invite-only, manual approval)
- Production Supabase schema (from spike, hardened)
- Production API deployment with endpoints: `GET /projects` (client-scoped)
- Production sync service running on schedule (every 5-15 min)
- Monitoring: sync failure alerting (email/Slack on consecutive failures), API error logging, external uptime check
- Admin onboarding process documented: how to add a new client (create Wix member + link to company via Caflou or admin page, depending on Phase 0 outcome)
- Active Projects portal page with: project name, status, assigned manager, last synced timestamp
- "Last synced at" indicator on the page

**Dependencies:** Phase 0 must be complete. All open questions resolved.

**Success criteria:**
1. Real client logs in and sees only projects from their linked companies
2. No unlinked company's projects are visible (isolation verified — tested by logging in as two different clients with different company links)
3. Project data refreshes within 15 minutes of a change in Caflou
4. "Last synced" timestamp is accurate
5. Admin can onboard a new client in under 10 minutes following the documented process
6. All data requests flow through the API — no direct Supabase calls from Wix in production
7. A simulated sync failure triggers an alert to the admin
8. API errors are logged with enough context to diagnose issues
9. External uptime check is active and alerts on downtime

---

## Phase 2: Orders

**Goal:** Clients can place orders with auto-filled identity and auto-calculated pricing.

**Requirements:** ORD-01, ORD-02, ORD-03, ORD-04, ORD-05, ORD-06, ORD-07

**Estimate:** 7–12 h

**What gets built:**
- API endpoints: `GET /pricing` (client-scoped rate lookup), `POST /orders` (validates, looks up rate server-side, creates task/project in Caflou)
- Order form page: read-only auto-filled name/company/email, page count input, live price display
- Order submission: validated by the API and created directly in Caflou (Caflou is the source of truth — no independent orders table in Supabase)
- Error handling: clear error message if Caflou is unreachable, client can retry
- Sync service brings orders back from Caflou as part of normal sync cycle (for portal visibility if needed later)

**Dependencies:** Phase 1 complete. Pricing sync from Caflou validated in Phase 0. Caflou write capability validated in Phase 0.

**Success criteria:**
1. Name, company, and email are pre-filled and non-editable
2. Total price = pages x client rate, calculated in real time as pages are entered
3. Two different clients see different rates (pricing isolation verified)
4. Submitted order creates a task/project in Caflou with correct client, rate, and total
5. Order submission goes through the API — rate is looked up server-side (not trusted from the form)
6. Order is acknowledged to the client (confirmation UI, at minimum)
7. If Caflou is unreachable, client sees an error and can retry (submission does not silently fail)

---

## Phase 3: Project Backups + Documentation & Handoff

**Goal:** Clients can access their Google Drive backups from the portal. Project is handed off to the business user.

**Requirements:** BACK-01, BACK-02, BACK-03

**Estimate:** 5–8 h

**What gets built:**
- Drive links synced from Caflou `custom_column_gdrive_link` attribute (should already be in `projects` table from Phase 0 sync)
- Backups data served through existing `GET /projects` API endpoint (or dedicated `GET /backups` if needed)
- Backups section on the projects page (or separate page): project name + Drive link
- Placeholder for projects with no Drive link attached
- Admin guide: how to onboard clients, set pricing, manage members, update Caflou tags
- Technical documentation: Supabase schema, API endpoints, sync logic, Wix backend functions, troubleshooting guide
- Handoff walkthrough with business user

**Prerequisites (business-side, not dev):** Team must be consistently attaching Google Drive links to projects in Caflou before this phase can deliver full value. This is a process requirement, not a technical one.

**Dependencies:** Phase 1 complete. Drive link workflow adopted by team.

**Success criteria:**
1. Projects with a Drive link show a clickable link that opens Google Drive
2. Projects without a Drive link show a clear placeholder (not an error)
3. Business user can onboard a new client without developer help, following the admin guide
4. Business user can update portal layout/design in Wix editor without touching code
5. Developer can diagnose a sync failure using the troubleshooting guide

---

## Requirement Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| AUTH-01 | Phase 0 | Pending |
| AUTH-02 | Phase 0 | Pending |
| AUTH-03 | Phase 0 | Pending |
| AUTH-04 | Phase 0 | Pending |
| AUTH-05 | Phase 0 | Pending |
| AUTH-06 | Phase 0 | Pending |
| SEC-01 | Phase 0 | Pending |
| SEC-02 | Phase 0 | Pending |
| SEC-03 | Phase 0 | Pending |
| SEC-04 | Phase 0 | Pending |
| SYNC-01 | Phase 0 | Pending |
| SYNC-02 | Phase 0 | Pending |
| SYNC-03 | Phase 0 | Pending |
| SYNC-04 | Phase 0 | Pending |
| SYNC-05 | Phase 0 | Pending |
| SYNC-06 | Phase 0 | Pending |
| MON-01 | Phase 1 | Pending |
| MON-02 | Phase 1 | Pending |
| MON-03 | Phase 1 | Pending |
| PROJ-01 | Phase 1 | Pending |
| PROJ-02 | Phase 1 | Pending |
| PROJ-03 | Phase 1 | Pending |
| PROJ-04 | Phase 1 | Pending |
| ORD-01 | Phase 2 | Pending |
| ORD-02 | Phase 2 | Pending |
| ORD-03 | Phase 2 | Pending |
| ORD-04 | Phase 2 | Pending |
| ORD-05 | Phase 2 | Pending |
| ORD-06 | Phase 2 | Pending |
| ORD-07 | Phase 2 | Pending |
| BACK-01 | Phase 3 | Pending |
| BACK-02 | Phase 3 | Pending |
| BACK-03 | Phase 3 | Pending |

**Coverage:** 33 v1 requirements | 33 mapped | 0 unmapped ✓

---
*Created: 2026-03-08*
