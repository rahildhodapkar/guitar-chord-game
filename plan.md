# Plan: Guitar Chord Rhythm Game

## Scope
- **Game (endless/survival):** chords flash, tempo ramps, run ends on miss. Client detects live; server re-scores recording for authoritative leaderboard number.
- **Practice mode:** same flashing chords, fretboard renders highlighted shape per chord. No scoring. Shapes from music-theory engine + curated voicing table.
- **Fretboard chord-ID tool:** click frets → identify chord. Client-only.
- **Profile:** accounts, run history/stats, custom fretboard tuning.

## Stack
| Layer | Choice |
|---|---|
| Frontend | Vite + React + TypeScript |
| Frontend state | TanStack Query + TanStack Router |
| Typed API client | `openapi-typescript` + `openapi-fetch` (generated from Go OpenAPI spec) |
| API | Go — REST+OpenAPI, auth, job orchestration, SSE |
| Audio worker | Python — DSP (now), ML (Phase 6) |
| Go↔Worker queue | Redis (job queue + pub/sub for SSE status) |
| Database | PostgreSQL — sqlc + goose |
| Object storage | S3-compatible; MinIO locally |
| Auth | Keycloak (self-hosted OIDC) |
| Deploy | Docker Compose local → Fly.io/Render; GitHub Actions CI/CD |

## Scoring (hybrid)
- **Client:** `AudioWorklet` + pitch/chroma → live hit/miss + miss-detection to end the run. Provisional score shown in UI.
- **Server:** on run submit, Go issues a presigned upload URL; Python worker runs: onset detect → CQT/chroma → per-note presence → compare vs expected chord tones at the server-issued timestamps.
- Metrics: note correctness, intonation (cents/string), per-string clarity (muted/dead/buzz), strum evenness, timing accuracy. Per-chord scores → aggregate. Low confidence → explicit flag, not a fake number.
- Server score is the only number that hits the leaderboard.

## Anti-cheat (baseline-plus)
- Server issues randomized chord sequence + timestamps per session. Re-score verifies chords land at those times → defeats pre-recorded uploads.
- Audio retained per run. Rate limits + anomaly flags on submission endpoint.
- Deep anti-cheat (device attestation, replay fingerprinting) = designed-in seam, Phase 6 / "future work."

## Leaderboards
- Global all-time, daily/weekly/seasonal, per-difficulty/per-song. Postgres ranked queries + pagination. Optional Redis sorted sets for hot boards later.
- Optional WebSocket: live leaderboard updates + presence only (never scoring).

## Phases
| # | What | Dependency |
|---|---|---|
| 0 | Monorepo, Docker Compose (postgres/redis/minio/keycloak), CI gates | — |
| 1 | Music-theory lib + voicing table + real-time chord-match engine (`AudioWorklet` + CQT). Unit tested. | 0 |
| 2 | Game loop UI (flashing chords, tempo ramp, miss detect, provisional score) + practice mode (fretboard shape overlay) + fretboard chord-ID tool. Local-only. | 1 |
| 3 | Go API: Postgres schema+migrations, Keycloak OIDC, profile + custom tuning CRUD, OpenAPI spec → generated TS client wired into TanStack Query | 0 |
| 4 | Scoring pipeline: server-issued sequence → presigned upload → Redis job → Python worker re-score → leaderboards. SSE for job status. | 2, 3 |
| 5 | Hardening: baseline-plus anti-cheat, observability (structured logs/metrics/traces), OWASP pass, Playwright E2E, cloud deploy + CI/CD. | 4 |
| 6 (stretch) | ML chord model in worker (DSP fallback behind interface), deep anti-cheat seams, friends leaderboard. | 5 |

## Verification
- **Unit:** chord-ID (table-driven), pitch detector vs synth tones, live chord-match logic, DSP grader vs labeled audio fixtures.
- **Integration:** Go + Postgres (testcontainers), auth routes, session-issue → upload → re-score → leaderboard insert, ranking/pagination.
- **E2E (Playwright):** play a run with a fixture audio clip, score lands on leaderboard.
- **Manual:** real guitar run — scoring feels fair and consistent.

## Open decisions
1. Auth: **Keycloak** (rec) vs Clerk/Auth0.
2. Queue: **Redis** (rec) vs NATS.
3. Go data layer: **sqlc + goose** (rec) vs GORM.
4. Leaderboard store: **Postgres only** (rec) vs + Redis sorted sets.
5. ML: **DSP first, ML in P6** (rec) vs ML earlier.
