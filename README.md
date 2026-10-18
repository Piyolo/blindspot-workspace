# BlindSpot

Rebuild and upgrade of the BlindSpot capstone — an Android app (Flutter) that helps blind and visually impaired users perceive household objects, plus a FastAPI backend.

- On-device object detection (existing YOLOv8n TFLite model, 14 classes) with spoken announcements + approximate distance and bounding boxes
- Always-on bilingual (English / Filipino) voice commands — no push-to-talk
- Volume buttons reserved exclusively for emergency calls (5 s cancelable countdown, double-press to call now)
- Accounts via FastAPI + Neon Postgres; core features work offline

## Layout

| Path | Contents | GitHub repo (created in Phase 0) |
|---|---|---|
| `app/` | Flutter Android app | `Piyolo/blindspot-app` |
| `backend/` | FastAPI service (Render + Neon) | `Piyolo/blindspot-api` |
| `docs/` | Design spec, implementation plan | `Piyolo/blindspot-workspace` (this repo) |

## Docs

- Design spec: [`docs/superpowers/specs/2026-10-17-blindspot-rebuild-design.md`](docs/superpowers/specs/2026-10-17-blindspot-rebuild-design.md)
- Implementation plan: added in Phase 0 planning (`docs/superpowers/plans/`)

## Status

- [x] Design approved — 2026-10-17
- [x] Phase 0 — scaffold, assets, theme, ApiClient, backend skeleton, live API, repos — 2026-10-18
- [ ] Implementation phases 1–6 (see spec §15)
- Live API: https://blindspot-api-dkch.onrender.com (Render, free plan, Singapore)

## Notes

- Original capstone manuscript and reference Android app are archived separately under OneDrive (`Academic Documents/BlindSpot/`); the original app is kept untouched.
- Never commit secrets. Neon connection string and JWT secret live only in Render environment variables and local `.env` (gitignored).
