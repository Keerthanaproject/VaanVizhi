# VaanVizhi — Climate Alert & Emergency Response Coordination

**VaanVizhi** (வான்விழி, "eye on the sky") is an AI-assisted, multilingual (Tamil/English), offline-first platform that connects **disaster authorities**, **citizens** and **field workers**:

> Authority drafts an alert → selects regions → human approval → publish → citizen PWA receives it live (+ simulated SMS) → citizen acknowledges and requests help (works offline) → field team accepts → everyone sees the update in real time.

⚠️ **Prototype for demonstration — not an official emergency service.** All regions, alerts, people, teams, shelters and deliveries are synthetic **DEMO** data. No real SMS is sent unless you configure a provider *and* enable the safety latch *and* recipients have consented.

---

## Quick start (Windows / macOS / Linux)

Requirements: **Node.js ≥ 22.13** (uses built-in `node:sqlite`; tested on Node 24). No database server needed for development.

```bash
npm run install:all          # installs root, server and web dependencies
cp server/.env.example server/.env   # optional: defaults work without editing
npm run db:migrate           # create tables (SQLite file: server/data/vaanvizhi.db)
npm run db:seed              # seed DEMO data + publish one DEMO alert
npm run dev                  # API http://localhost:4000 + web http://localhost:5173
```

Open **http://localhost:5173** → choose a demo role. Open each role in a **separate browser window/tab** (sessions are per-tab) to watch real-time propagation.

| Command | What it does |
|---|---|
| `npm run dev` | API (tsx watch) + Vite dev server |
| `npm run build` | Server typecheck + production PWA build (`web/dist`) |
| `npm start` | API serving the built PWA at http://localhost:4000 (service worker active) |
| `npm run demo:reset` | Wipe and re-seed DEMO data (also available in the authority sidebar: **Reset demo**) |
| `npm test` | API acceptance tests (API must be running) |
| `npm run e2e` | Browser end-to-end + offline test using installed Edge/Chrome (needs `npm start` or `npm --prefix web run preview`) |

> The service worker (offline guide, cached app shell) only exists in the **production build**. For the offline demo use `npm run build && npm start` and open http://localhost:4000.

## Demo accounts (DEMO_MODE=true)

| Role | User | Lands on |
|---|---|---|
| Authority | Demo Control Room Officer | `/authority` command centre |
| Citizen | Kavya (Velachery) | `/citizen` mobile PWA |
| Field worker | Arun — Team Alpha (T-01) | `/field` |

One-click demo login issues an HMAC-signed 12-hour token. **This is not production authentication** (no passwords/OTP). Authorization is enforced server-side for every endpoint.

## Judge demo script (≈5 min)

1. **Authority** → *New alert* → Flood → select *Velachery–Pallikaranai* on the map (audience preview appears) → Orange → *Generate Tamil & English draft* → Submit → **Review & approve** (explicit checkbox) → **Publish now**.
2. **Citizen** window: alert appears instantly with a toast. Open it → *Listen* (text-to-speech) → **I've read this alert** → see *SMS simulator* (messages labelled SIMULATED).
3. Citizen → **Request Help** → Evacuation → tap map for location → Send. Status shows *Received by VaanVizhi server*.
4. **Field** window: request pops up live → open → **Accept for T-01** → citizen sees *Worker accepted*; authority overview updates.
5. **Offline**: in the citizen tab tick *Simulate offline (demo)* (or DevTools → Network → Offline). Guide and last alerts remain, with "Saved copy … may be outdated". Request help → **"Saved on this device — not yet sent to responders."** Untick → it syncs once and appears on field/authority dashboards.
6. Authority → alert → **Find unreached** → *Reachability & outreach* shows prioritised follow-up tasks; **Issue correction** (v2), **Cancel** with reason.
7. Authority → **Integrations** shows what is real vs simulated.

## Architecture

```
web/  React 19 + TypeScript + Vite + Tailwind v4 · React Router · TanStack Query · React Hook Form + Zod
      Leaflet/OSM · Socket.IO client · Dexie (IndexedDB) outbox · vite-plugin-pwa (Workbox)
server/ Node + Express 5 + TypeScript (tsx) · Socket.IO · Zod validation · node:sqlite persistence
      adapters/  sms (simulator | Twilio) · ai (Anthropic | rule-based) · routing (OSRM | straight-line) · weather (CAP/RSS | none)
docs/postgres-schema.sql  PostGIS production schema
```

- **Persistence:** SQLite file via Node's built-in `node:sqlite` (no native build, no server). PostgreSQL/PostGIS was not available in the build environment; `docs/postgres-schema.sql` is the production target (not executed here).
- **Real-time:** Socket.IO rooms per role/user. Every write is committed *before* the event is emitted. The header badge shows *Live* only when the socket is actually connected; TanStack Query polls every 60 s as fallback.
- **Offline queue:** requests and field reports are written to IndexedDB **first**, with a client-generated UUID that is also the server's idempotency key. `POST /api/sync` replays ops; duplicates return the existing record (`duplicate: true`). Retries run on `online` events, app launch, every 20 s, and the manual *Sync* button. Stale edits are rejected with **409** using optimistic `version` checks. Background Sync API is **not** relied on — sync happens while the app is open.
- **Cached reads:** alerts, shelters, requests, map layers are stored in IndexedDB with their fetch time, so offline views always show freshness.

## Simulated vs real capabilities

| Capability | Status in this build |
|---|---|
| Alert workflow (draft → approval → publish → correct/cancel/expire, versions, audit) | **Real** |
| Shared DB, role-based API authorization, real-time updates | **Real** |
| Offline guide, cached alerts, IndexedDB queue, idempotent sync | **Real** (verified in headless Edge) |
| Road routing | **Real** via public OSRM demo server (`router.project-osrm.org`, light non-commercial use only). Falls back to a labelled straight-line estimate. Routes are never described as safe. |
| Map tiles | **Real** OpenStreetMap tiles (online only; no offline tile download per OSM policy) |
| SMS | **Simulated** by default (status `simulated`, never `delivered`). Twilio adapter implemented, untested without credentials. |
| AI drafting / assistant / categorisation | **Deterministic fallback** by default (clearly labelled). Anthropic adapter activates with `ANTHROPIC_API_KEY`; not verified in this environment. |
| Official warnings (IMD / NDMA SACHET) | **Not integrated** — no verified access. Optional `CAP_FEED_URL` shows items as reference only, never auto-published. |
| Regions, shelters, teams, recipients, team locations | **Synthetic DEMO data**; team positions are seeded/manually reported, not live GPS |
| Web Push, Voice/IVR | **Not implemented** |
| Authentication | **Demo sessions only** |

## Configuration (`server/.env`)

See `server/.env.example`. Key variables: `SESSION_SECRET`, `DEMO_MODE`, `ANTHROPIC_API_KEY` / `AI_MODEL`, `SMS_PROVIDER=twilio` + `TWILIO_*` + `SMS_ALLOW_REAL_SEND`, `OSRM_URL`, `CAP_FEED_URL`, `REQUIRE_SECOND_APPROVER`.

**SMS:** real sending requires all of: Twilio credentials, `SMS_ALLOW_REAL_SEND=true`, recipient `sms_consent=1`, and a real E.164 number. Point Twilio's status callback to `POST /api/sms/status-callback`; only carrier-confirmed callbacks mark `delivered`. Respect DLT sender/template registration rules in India.

**Routing:** for production self-host OSRM (or Valhalla/GraphHopper behind the same adapter) and set `OSRM_URL`.

## API (all under `/api`, JSON `{ data }` or `{ error, details }`)

| Area | Endpoints |
|---|---|
| Auth | `POST /auth/demo-login {role}`, `GET /auth/me`, `GET /auth/demo-users` |
| Regions/map | `GET /regions`, `GET /map/layers`, `GET /recipient-groups`, `POST /audience-preview` |
| Alerts | `GET /alerts`, `GET /alerts/:id`, `POST /alerts`, `PUT /alerts/:id`, `POST /alerts/:id/{submit,approve,publish,correct,cancel,expire}`, `POST /alerts/:id/ack`, `POST /alerts/:id/received`, `GET /alerts/:id/deliveries`, `POST /ai/draft-alert` |
| Requests | `GET /requests`, `GET /requests/:id`, `POST /requests` (idempotent by UUID), `POST /requests/:id/transition {action, expectedVersion}` |
| Sync | `POST /sync {ops:[{opId, kind, payload}]}` |
| Teams | `GET /teams`, `PATCH /teams/:id` |
| Reports | `GET /reports`, `POST /reports`, `POST /reports/:id/verify` |
| Shelters/resources | `GET /shelters`, `PATCH /shelters/:id`, `GET /resources`, `POST /resources/:id/adjust`, `GET /resources/:id/movements` |
| Outreach | `GET /outreach`, `POST /outreach/scan`, `PATCH /outreach/:id`, `GET /field-workers` |
| Ops | `GET /dashboard/summary`, `GET /audit`, `GET /integrations`, `GET /feed`, `POST /route`, `POST /assistant`, `GET /sms/inbox`, `POST /sms/status-callback`, `POST /demo/reset`, `GET /health` |

Request lifecycle actions: `triage, assign, accept, decline (reason), en_route, assisted, close, escalate, cancel`. Socket events: `alert:published|changed`, `request:new|updated`, `teams:changed`, `report:new|updated`, `shelters:changed`, `resources:changed`, `outreach:changed`, `deliveries:changed`, `ack:new`, `demo:reset`.

## Testing

- `npm test` — 12 API/real-time acceptance tests (T1–T7, T10–T13, data integrity). Resets demo data before and after.
- `npm run e2e` — 12 browser steps in headless Edge (`BROWSER=chrome` for Chrome): UI alert composition/approval/publish, live citizen receipt, ack, live field receipt, accept, **offline guide + cached alert with network disabled, IndexedDB "not sent" request, reconnect sync, no duplicate on replay**, RBAC redirect. Screenshots in `web/e2e/screenshots`.

## Deployment

1. `npm run install:all && npm run build`
2. Set `server/.env`: strong `SESSION_SECRET`, `DEMO_MODE=false` for anything beyond a demo, `CORS_ORIGIN` to your domain.
3. `npm start` serves API + PWA on `PORT` (default 4000). Put it behind HTTPS (required for service workers, geolocation and installability on non-localhost origins), e.g. Caddy/Nginx; enable WebSocket upgrade for `/socket.io`.
4. Persist `server/data/` (SQLite) or migrate to PostgreSQL/PostGIS using `docs/postgres-schema.sql`.
5. Before real use: replace demo login with real identity (OTP/SSO), self-host routing, obtain official feed access, register SMS sender/templates, add retention jobs for assistance records.

## Security & privacy notes

Server-side RBAC on every route · Zod input validation · rate limits on login, publish, requests, sync, assistant, routing · audit log for approvals, publishing, corrections, cancellations, assignments, resource and shelter changes · provider keys only on the server · minimal citizen data (type, count, optional consented location, short note) · demo recipients have masked, non-dialable numbers.
