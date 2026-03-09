# Client Portal — Stakeholder Brief

> **Status**: Refined plan v1.0
> **Date**: 2026-03-08
> **Full technical plan**: .planning/ROADMAP.md

---

## What We Are Building

A private portal on the existing Wix website where each client logs in and sees only their own data:

1. **Active Projects** — pulled automatically from Caflou, refreshed every 5–15 minutes
2. **Order Form** — pre-filled with their details, auto-calculated pricing based on their negotiated rate
3. **Project Backups** — direct links to their Google Drive folders

Access is invite-only. The business admin creates client accounts and links them to one or more companies. Each client sees only their linked companies' data — no access to unlinked companies is possible.

---

## How It Works

The portal is built in four layers:

```
Caflou (your CRM — source of truth for everything)
    ↓  automated sync, every 5–15 minutes (projects, pricing, Drive links)
Database (holds a local copy — read cache only)
    ↓
API (validates who is asking, returns only their data, sends orders to Caflou)
    ↓                                                         ↑
Wix (the presentation layer — pages, login, design)     orders go directly
                                                         to Caflou via API
```

**Caflou** remains the single source of truth. Pricing, project data, Drive links, and orders all live there. Nothing needs to be re-entered elsewhere. When a client submits an order, it goes directly to Caflou — not to a separate database.

**The database** (an external service called Supabase) acts as a fast local copy for reading data. The portal reads from it, so page loads are instant and not dependent on Caflou's API being available at that moment. It does not store orders independently.

**The API** is a small backend service that sits between the database and the website. When a client views a page, the website asks the API for data. The API checks who the client is, then queries the database for only that client's records. The database credentials never leave the API — the website cannot query the database directly.

**Wix** handles what it does best: the visual design, client login, and displaying data. The business user can continue updating the layout, colors, and content in the Wix editor without any developer involvement.

### Why Not Pure Wix?

Wix has a hard 14-second time limit on all background operations and restricts automated data refreshes to once per hour on standard plans. These constraints plus the unnecessary complexity of Wix developer tools make a reliable sync impossibly complex in Wix alone. Moving the sync to an external service removes both limitations — it can run every 5 minutes with no timeout risk.

### Why a Separate API Layer?

Without the API layer, the website would need to hold database credentials and query the database directly. This creates several security risks:

1. **Credential exposure** — Database access keys would be stored inside Wix, a platform we don't fully control. If those keys leak (through a Wix vulnerability, misconfiguration, or browser inspection), an attacker could access every client's data directly.
2. **No safety net** — Data isolation would depend entirely on every query in the website code being written correctly. A single missed filter (e.g., forgetting a `WHERE client_id = ?` clause) would expose other clients' data. There would be no second layer to catch the mistake.
3. **No audit trail** — With direct database access from the website, there's no centralized place to log who accessed what data and when.
4. **Harder to secure over time** — As features are added (orders, backups, future functionality), each new query is another place where a filtering mistake could leak data.

The API layer eliminates all of these. The website never touches the database. The API validates every request, applies the correct filter, and returns only what that specific client should see. Even if a bug is introduced on the website side, the API will not return another client's data.

### How Clients Are Linked to Their Data

Three systems are involved (Wix login, database, Caflou), and they are linked through two identifiers:

- A **client ID** — a unique code assigned to each company. It appears in the database's company registry (alongside the matching Caflou tag) and on every project row (stamped by the sync service).
- A **member ID** — the Wix login account identifier. The database stores which member is linked to which company(ies).

One person can be linked to **multiple companies** — they will see projects from all their linked companies. Multiple people can be linked to the **same company** — they all see the same projects.

When a client loads a page, the website sends their login session to the API. The API verifies the session with Wix, looks up which companies that person is linked to, and queries the database for everything matching those companies. Verified server-side, not trusted from the browser.

An admin sets this up once per client:

1. Create the client's login account in Wix
2. In the database, link that account to one or more companies
3. Ensure each company's Caflou tag is registered in the database

If a company renames in Caflou, only the database company registry needs updating — the client's login and all their project rows stay intact, because they reference the stable client ID, not the tag string.

---

## What Needs to Be in Place Before Development Starts

These are business-side prerequisites, not technical ones. Development can begin without them, but the portal will not work correctly until they are done.

| Prerequisite | Why It Matters | Owner |
|---|---|---|
| Caflou projects are tagged with the exact client company name | This is how the portal identifies which projects belong to which client | Business team |
| Google Drive backup links are attached to projects in Caflou | This is how the portal shows backup files — no separate Drive integration is needed | Business team |
| Per-client pricing is stored in Caflou (as a custom field on the company) | The order form calculates prices automatically from this field | Business team |
| A Wix Business plan (or equivalent) is active | Required to enable member login and the portal pages | Business decision |

Tags and Drive links must be applied **consistently** going forward. If a project is missing a tag, it will not appear in the portal. If a Drive link is missing, the portal will show a placeholder.

---

## Phases

Estimates assume a developer working with an AI coding agent. The AI handles most scaffolding, boilerplate, and repetitive code. The developer focuses on integration, testing, and decisions that require human judgment (Caflou API exploration, Wix platform quirks, production deployment).

### Phase 0 — Validation Spike

**Goal:** Answer all open technical questions before committing to the full build. No production code is written — only a working proof-of-concept.

| Task | Estimate |
|---|---|
| Set up Wix development site with Velo | 1–2 h |
| Test Caflou API: auth, project listing, tag format, custom fields, pagination, task creation (write) | 3–5 h |
| Set up database and deploy API proof-of-concept | 2–3 h |
| Build a sync proof-of-concept (Caflou → database → API → Wix test page) | 3–4 h |
| Verify the full identity chain: logged-in client → API → their projects only | 1–2 h |
| Document all findings and choose the sync service and API host | 1 h |
| **Total** | **11–17 h** |

**Output:** A working proof-of-concept where a test client logs in and sees only their own projects, with data flowing through the secure API layer. All open technical questions answered. Sync service and API host chosen. Phase 1 estimates can be tightened after this.

**Biggest risk:** Caflou's API behavior is unknown until tested. If tag filtering is not supported by the API, all projects must be fetched and filtered in code (adds complexity, not a blocker). Phase 0 exists specifically to surface risks like this. The Caflou exploration tasks are the hardest to accelerate with AI — they require hands-on experimentation with an undocumented API.

---

### Phase 1 — Active Projects

**Goal:** Real clients can log in and see their live project list.

| Task | Estimate |
|---|---|
| Production Wix member area (invite-only login, manual approval) | 2–3 h |
| Production database schema and data security setup | 1–2 h |
| Production API deployment with session validation and client-scoped endpoints | 2–3 h |
| Production sync service running on schedule (every 5–15 min) | 2–4 h |
| Active Projects page (project name, status, manager, last synced time) | 3–5 h |
| Basic monitoring setup (sync health, API errors, uptime check) | 1–2 h |
| Admin onboarding guide (how to add a new client without a developer) | 1–2 h |
| Testing with real client data (including cross-client isolation verification) | 2–3 h |
| **Total** | **14–24 h** |

**Output:** Clients can log in and see their projects. Business user can onboard new clients independently following the written guide. Data refreshes automatically. All data access goes through the secure API. Basic monitoring alerts on sync failures and API errors.

---

### Phase 2 — Orders

**Goal:** Clients can place orders without re-entering their details, with automatic price calculation.

| Task | Estimate |
|---|---|
| Order API endpoint (server-side validation, rate lookup, Caflou task creation) | 2–3 h |
| Order form page (auto-fill, page count input, live price display) | 2–4 h |
| Error handling (Caflou unreachable → clear error, client retries) | 1–2 h |
| Testing (including pricing isolation between clients) | 1–2 h |
| **Total** | **6–11 h** |

**Output:** Clients can submit orders. The form knows who they are and what their rate is. Orders are validated by the API and created directly in Caflou — keeping Caflou as the single source of truth. If Caflou is temporarily unreachable, the client sees an error and can retry.

---

### Phase 3 — Project Backups + Handoff

**Goal:** Clients can access their Google Drive backups from the portal. Project is handed over to the business user.

| Task | Estimate |
|---|---|
| Drive links section on the projects page | 1–2 h |
| Handling projects with no Drive link (placeholder, not an error) | 0.5–1 h |
| Admin and technical documentation | 2–3 h |
| Handoff walkthrough with business user | 1–2 h |
| **Total** | **5–8 h** |

**Output:** Clients see their Drive links next to each project. Business user has full documentation to manage the portal and onboard new clients independently.

**Prerequisite:** The team must be consistently attaching Google Drive links to projects in Caflou before this phase delivers full value. This is a process change, not a technical one, and can run in parallel with earlier phases.

---

## Total Estimate

| Phase | Low | High |
|---|---|---|
| Phase 0 — Spike | 11 h | 17 h |
| Phase 1 — Active Projects | 14 h | 24 h |
| Phase 2 — Orders | 6 h | 11 h |
| Phase 3 — Backups + Handoff | 5 h | 8 h |
| **Total** | **36 h** | **60 h** |

The low end assumes: Caflou API supports tag filtering and task creation directly, tags are consistent, simple page design, Drive links already in use.

The high end assumes: Caflou API requires full-fetch and manual filtering, complex design, Caflou task creation has unexpected complexity, need to establish the Drive link workflow from scratch.

Estimates assume AI-assisted development. The main bottleneck is not writing code — it's understanding how Caflou's API behaves (Phase 0) and verifying real-world behavior with production data (Phase 1).

---

## Open Questions

These should be resolved before or during Phase 0:

| # | Question | Impact |
|---|---|---|
| 1 | Does Caflou API support filtering projects by tag? | If no: +2–3 h in sync logic |
| 2 | What is the exact format of Caflou tags? | Tag in database must match exactly |
| 3 | Is pricing stored as a Caflou custom field on the company? | If no: pricing sync needs a different approach |
| 4 | Can the API create tasks/projects in Caflou? (write capability) | Orders go directly to Caflou — must be validated in Phase 0 |
| 5 | How many active projects exist across all clients? | Over ~200: pagination handling needed in sync service |
| 6 | Where should the API layer be hosted? | See options below — decided during Phase 0 |
| 7 | Which sync service should run the Caflou → database pipeline? | See options below — decided during Phase 0 |

### Sync Service Options (decided in Phase 0)

The sync service runs on a schedule (every 5–15 minutes), pulls data from Caflou, and writes it to the database. It's a simple pipeline — no user-facing traffic, no complex logic. Two options:

| Option | What It Is | Pros | Cons |
|---|---|---|---|
| **Make.com** | Visual no-code automation platform | Drag-and-drop flow builder — easy to inspect, modify, and hand off to a non-developer. Built-in error notifications and retry logic. Free tier included. | Per-operation pricing at scale. Another platform to manage. Less control over edge cases. |
| **Cloud Run Job (Google)** | A containerized script that runs on a schedule | Full code control (TypeScript/Python). Version-controlled, testable, no operation limits. Scales to zero — only runs when triggered. Free tier covers ~50,000 invocations/mo. | Requires a developer to maintain. No visual interface for the business user. More initial setup. |

**Recommendation:** If the business user wants to inspect or adjust the sync logic independently, choose **Make.com**. If the developer (or AI agent) will maintain the sync long-term and code control matters more, choose **Cloud Run Jobs**. Both are free at this project's scale. The choice can be made during Phase 0 after testing the Caflou API — the complexity of the API response will influence which option is more practical.

### API Hosting Options (decided in Phase 0)

The API is a small set of endpoints (3–4 functions). It does not need a full server. The best fit depends on cost, simplicity, and how well it integrates with Supabase:

| Option | What It Is | Pros | Cons |
|---|---|---|---|
| **Supabase Edge Functions** | Serverless functions hosted alongside the database (Deno runtime) | Zero extra infra — deployed with Supabase. Free tier includes 500K invocations/mo. Closest to the data. | Deno/TypeScript only. Newer platform. |
| **Cloudflare Workers** | Serverless functions on Cloudflare's edge network | Very fast (runs at edge). Generous free tier (100K requests/day). Mature platform. | Separate service to manage. Connects to Supabase over network. |
| **Cloud Run (Google)** | Containerized service on Google Cloud | Full control, any language/framework. Scales to zero. | More setup. Free tier is generous but has limits. Slight cold-start latency. |
| **Vercel / Netlify Functions** | Serverless functions tied to a deployment platform | Easy deployment. Good DX. | Adds a platform dependency. Free tiers vary. |

**Recommendation:** Start with **Supabase Edge Functions** in Phase 0. They deploy alongside the database, require no extra infrastructure, and are free at this project's scale. If they prove limiting during the spike, switch to Cloudflare Workers — the migration is straightforward since both use standard web APIs.

---

## Monitoring

The portal includes basic monitoring from Phase 1 onward to catch problems before clients notice them:

- **Sync health** — if the sync service fails repeatedly, the admin receives an alert (email or Slack). Clients continue to see the last synced data, but the admin knows it's stale.
- **API errors** — failed API requests are logged with context (which endpoint, what error). If the error rate spikes, the admin is alerted.
- **Uptime check** — an external service pings the API on a schedule. If it's down, the admin is notified.

This uses free-tier tooling (Supabase logs, a free uptime monitor like UptimeRobot or Better Stack). No additional cost at this project's scale.

---

## Ongoing Costs After Launch

| Item | Estimated Monthly Cost | Notes |
|---|---|---|
| Wix plan | $0 additional | Current Business plan covers member login and portal pages |
| Database (Supabase) | $0 – $25/mo | Free tier: 500 MB storage, 2 GB transfer, 500K Edge Function invocations. Sufficient for this project's volume. Paid Pro plan ($25/mo) only if usage grows. |
| API layer | $0 (included with Supabase) | If using Supabase Edge Functions — no separate cost. If Cloudflare Workers: free tier covers 100K requests/day. Either option is $0 at this scale. |
| Sync service | $0 – $10/mo | Make.com: free tier (1,000 ops/mo), paid from ~$10/mo. Cloud Run Jobs: free tier (~50K invocations/mo), effectively $0 at this scale. Either option works. |
| Monitoring | $0 | Free tiers of Supabase logs + UptimeRobot or Better Stack |
| **Total estimated** | **$0 – $35/mo** | All services have free tiers that cover this project's expected volume. Costs only arise if usage significantly exceeds expectations. |

Developer maintenance is as-needed — sync logic, API endpoints, and Wix Velo code are documented and straightforward to hand off.

---

## Recommended Next Step

**Approve Phase 0 (Spike).** 11–17 hours of work answers all the critical questions: Does Caflou's tag filtering work? What do the tags look like? How does the pricing field work? Which API host fits best? The output is a working proof-of-concept with data flowing securely from Caflou through the API to a Wix page. After Phase 0, estimates for the remaining phases can be confirmed with much higher confidence.
