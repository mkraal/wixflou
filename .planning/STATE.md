# Project State

## Project Reference

See: .planning/PROJECT.md

**Core value:** Each client logs in and immediately sees only their own company's live project data — without the business team manually sending updates, links, or pricing by email.
**Current focus:** Not started — ready for Phase 0

## Phase Status

| Phase | Name | Status | Plans |
|-------|------|--------|-------|
| 0 | Spike — Validate & Prove the Stack | Pending | — |
| 1 | Active Projects | Pending | — |
| 2 | Orders | Pending | — |
| 3 | Project Backups + Handoff | Pending | — |

## Current Phase

None — project initialized, no phase started yet.

## Notes

- Phase 0 is a spike: output is validated answers + working proof-of-concept, not production code
- Phase 0 validates the full secure chain: Wix → API → Supabase, plus Caflou sync, plus API host selection
- Phase 1 includes basic monitoring setup (sync health, API errors, uptime)
- Phase 3 has a business-side prerequisite: team must adopt the Caflou Drive link attachment workflow
- Orders go directly to Caflou via the API (Caflou is the source of truth for orders — no independent orders table in Supabase)
- Multi-company access supported: one member can be linked to multiple companies via `client_members` join table
- Member-to-company mapping mechanism is a Phase 0 question: Caflou-native (preferred) vs admin page in Wix (fallback)
- Development is AI-assisted — estimates reflect a developer working with an AI coding agent against the detailed plans

---
*Initialized: 2026-03-08*
