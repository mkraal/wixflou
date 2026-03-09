# Requirements: Client Portal (Wix + Supabase + API Layer)

**Defined:** 2026-03-08
**Core Value:** Each client logs in and immediately sees only their own company's live project data — without the business team manually sending updates, links, or pricing by email.

## v1 Requirements

### Authentication

- [ ] **AUTH-01**: Client can log in with email and password via Wix Members
- [ ] **AUTH-02**: Portal is invite-only — no self-registration
- [ ] **AUTH-03**: Client session persists across browser refresh
- [ ] **AUTH-04**: Client sees only their linked companies' data (enforced by the API layer — memberId resolved server-side, company access looked up from `client_members` table, scoped on every query)
- [ ] **AUTH-05**: Admin can create a new client account in Wix dashboard without developer involvement
- [ ] **AUTH-06**: Admin can link a Wix member to one or more companies without developer involvement (mechanism TBD in Phase 0 — Caflou-native preferred, admin page in Wix as fallback)

### Security

- [ ] **SEC-01**: All portal data access goes through the API layer — Wix frontend and `.jsw` backend never query Supabase directly
- [ ] **SEC-02**: API authenticates every request server-side before processing — either by verifying a Wix OAuth token or by validating a shared API key from the `.jsw` backend (mechanism determined in Phase 0)
- [ ] **SEC-03**: Database credentials (`service_role` key) are stored only in the API layer — never in Wix code, environment variables, or frontend
- [ ] **SEC-04**: API enforces `client_id IN (member's linked companies)` scoping on every data query as defense in depth — even if application logic has a bug, the API will not return data from unlinked companies

### Data Sync

- [ ] **SYNC-01**: External sync service runs every 5-15 minutes pulling from Caflou API
- [ ] **SYNC-02**: Sync pulls all active projects and upserts into Supabase keyed on Caflou project ID
- [ ] **SYNC-03**: Sync pulls per-client pricing from Caflou company custom attribute
- [ ] **SYNC-04**: Sync pulls Google Drive backup links from Caflou project custom attribute
- [ ] **SYNC-05**: Sync failure does not break the portal — last synced data remains visible
- [ ] **SYNC-06**: Each project row stores a `lastSyncedAt` timestamp

### Monitoring

- [ ] **MON-01**: Consecutive sync failures trigger an alert to the admin (email or Slack)
- [ ] **MON-02**: API errors are logged with context (endpoint, error type, clientId) for debugging
- [ ] **MON-03**: External uptime check pings the API on a schedule and alerts the admin if it is unreachable

### Active Projects

- [ ] **PROJ-01**: Client can view all active projects for their company
- [ ] **PROJ-02**: Each project displays: name, status, assigned manager, last synced timestamp
- [ ] **PROJ-03**: Projects are filtered by company — no projects from other clients visible
- [ ] **PROJ-04**: Page shows when data was last synced

### Orders

- [ ] **ORD-01**: Order form auto-fills client name, company, and email from Wix member profile
- [ ] **ORD-02**: Auto-filled fields are read-only — client cannot edit their own identity fields
- [ ] **ORD-03**: Client enters number of standard pages
- [ ] **ORD-04**: Total price is calculated live: pages x client's rate (rate fetched from API, sourced from Caflou via Supabase)
- [ ] **ORD-05**: Client can submit the order
- [ ] **ORD-06**: Submitted order is validated by the API and created as a task/project in Caflou (Caflou is the source of truth for orders — no independent orders table in Supabase)
- [ ] **ORD-07**: If Caflou is unreachable during order submission, the client sees a clear error message and can retry

### Project Backups

- [ ] **BACK-01**: Each project with a Drive link shows the link alongside the project name
- [ ] **BACK-02**: Clicking a Drive link opens Google Drive in a new tab
- [ ] **BACK-03**: Projects without a Drive link show a placeholder (link not yet available)

## v2 Requirements

### PM-Level Filtering

- **PM-01**: Each PM sees only projects assigned to them (not all company projects)
- **PM-02**: Requires Caflou to expose "assigned manager" field via API — verify in future spike

### Notifications

- **NOTF-01**: Client receives email confirmation after order submission
- **NOTF-02**: Business receives email/notification when new order is submitted

### Order Routing

- **ORD-V2-01**: Order status updates sync back from Caflou to portal

### Sync Enhancements

- **SYNC-V2-01**: Webhook from Caflou triggers immediate sync on project change (seconds-level freshness)
- **SYNC-V2-02**: Sync error monitoring dashboard (beyond basic alerting)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Self-registration | Invite-only by design — admin controls identity mapping |
| Payment processing / invoicing | Out of portal scope |
| File upload from client | Not requested |
| Chat / messaging | Not requested |
| Mobile app | Responsive web only |
| Multi-language support | Single market |
| Real-time sync (webhooks) | 5-15 min polling sufficient for v1 |
| Analytics dashboard for clients | Not requested |
| Separate Google Drive API integration | Drive links come from Caflou attachments — no direct Drive integration needed |

## Traceability

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

**Coverage:**
- v1 requirements: 33 total
- Mapped to phases: 33
- Unmapped: 0 ✓

---
*Defined: 2026-03-08*
