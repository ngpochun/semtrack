# SemTrack — Semester Progress Tracker for University Students

One platform to track your semester: assignment deadlines and group project progress, with a calendar view. Built for HKU students; the roadmap adds a Chrome extension that scrapes HKU Moodle course pages and auto-populates the calendar.

> **Status: early-stage prototype + design docs.** The web prototype is functional (in-memory demo data). The Chrome extension and backend are designed but not yet implemented.

## What's in this repo

| Path | What it is | Status |
|---|---|---|
| `semtrack-prototype.html` | Notion-style web app prototype: Overview, Assignments, Group Projects (tasks + milestones), Calendar. Single-file interactive fragment with linked state (checking a task updates progress everywhere). | ✅ Working prototype, browser-QA'd (desktop + mobile) |
| `docs/specs/product-spec.md` | MVP product spec: problem, scope (assignments + group projects), acceptance criteria, Phase 2 roadmap. | ✅ Complete (zh-TW) |
| `docs/architecture/backend-components.md` | Backend component design: lightweight monolith MVP; Phase 2 extension ingest, SHA-256 dedup file registry, PDF parse + LLM extraction workers, review queue. Data model + API draft. | ✅ Complete (zh-TW) |
| `docs/plans/implementation-plan.md` | File-level implementation plan (Next.js + Prisma + SQLite suggested), TDD steps, risks, rollback. | ✅ Complete (zh-TW) |
| `deliverables/product-strategy/prd-moodle-extension-2026-11-17.md` | Engineering-review PRD for the Moodle Chrome extension: one-click "Scrap and upload to SemTrack" → deadlines auto-added to calendar. Flows, edge cases, state machine, API contract, acceptance tests, decision log. | ✅ v0.1 (zh-TW); decisions D1/D2/D4/D5 answered by product owner, doc update pending |
| `semscan-prototype.html` | Archived earlier design direction (SemScan, highlighter-pen visual). Superseded by SemTrack. | ⚠️ Incomplete archive — kept as-is for design history |
| `output/visualizations/` | Published self-contained builds of the prototype (open directly in a browser). | ✅ Generated |
| `assets/screenshots/` | QA screenshots (desktop 1280px, mobile 390px). | ✅ Generated |

## Product decisions (confirmed)

- **Auto-add to calendar**: high-confidence extractions go straight to the calendar; low-confidence items go to a review queue.
- **First target**: HKU Moodle (`moodle.hku.hk`).
- **Reschedule handling**: out of scope for this phase.
- **Extraction test samples**: provided by the product owner.

## Roadmap

1. **MVP web app** (this prototype → real app): manual entry, assignments + group projects, derived calendar. Suggested stack: Next.js + Prisma + SQLite.
2. **Chrome extension (MV3)**: inject "Scrap and upload to SemTrack" on HKU Moodle course pages → collect course PDFs → dedup by SHA-256 → upload.
3. **Extraction pipeline**: PDF → text → LLM extraction (dates, type, confidence, source quote) → auto-calendar / review queue.

## Running the prototype

Open any file in `output/visualizations/` in a browser, or serve the repo root:

```bash
python3 -m http.server 8000
# then open http://localhost:8000/output/visualizations/<build>.html
```

All demo data is fictional and labeled as such. No backend, no persistence — state lives in memory.

## Not yet done

- Chrome extension implementation (shell + real scraping/upload)
- English UI translation toggle for the web prototype
- Backend implementation
- PRD v0.2 decision-log update

## License

No license file yet — all rights reserved by the author until one is added.