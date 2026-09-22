# Pulsar — Work Blocks: Coach & Athlete

**Source:** `entrega-equipe/MODEL.md`, `entrega-equipe/UML.md`
**Scope:** MVP (D6) — core + group dashboard, **without Strava**. Activities enter by file upload and manual entry only.
**Stack:** React Native (+ on-device DB) · API layer: NestJS + MySQL · Microservices: NestJS + MySQL · Messaging: Kafka + RabbitMQ.

Every block lists: tables touched, screens, server actions (gateway routes → service), messages, rules that must not be broken, and definition of done.

---

## Architecture baseline

### Topology

```
┌─────────────────────────── Mobile (React Native) ───────────────────────────┐   ┌── Garmin watch (Block 9) ──┐
│  Coach flows (role coach/admin)              Athlete flows (role athlete)   │   │ Pulsar data field (Monkey C)│
│  ─────────────────────────────────────────────────────────────────────────  │   │ samples run → /ciq/*        │
│  Local DB (SQLite) = read cache + outbox     Secure storage = tokens        │   │ (send-only, no data back)   │
└──────────────┬─────────────────────────────────────────────▲────────────────┘   └──────────────┬──────────────┘
               │ requests                                    │ responses · /sync?since= · push (FCM/APNs) │ HTTPS · device token
               │ HTTPS · JSON · JWT (access + refresh)       │ processed data: activities, sessions,       │ 204 / 401 only
               │                                             │ results, dashboards                         │
┌──────────────▼─────────────────────────────────────────────┴─────────────────────────────────────────▼─────┐
│  api-gateway  (NestJS)  — the "API layer" / BFF for the app (+ /ciq/* for the watch)                       │
│  · auth middleware → { userType, userId, assessoriaId, role }                                               │
│  · one route per screen need; composes calls to services                                                   │
│  · enriches IDs with names (only place personal data is joined in)                                          │
│  · MySQL `gateway_db`: refresh tokens, invites, rate-limit, idempotency keys, watch device bindings         │
│  · talks to services ONLY through the brokers — no internal HTTP                                           │
└──────────────────────────┬──────────────────────────────────────────────────▲──────────────────────────────┘
                           │ send()  request/response  (reads, writes the screen must confirm)                 │ consumes Kafka
                           │ emit()  commands          (uploads, long jobs → 202 Accepted)                     │ → push to app
┌──────────────────────────▼──────────────────────────────────────────────────┐  ┌────────────────────────────┴───────────────┐
│  RabbitMQ  — request/response + commands                                    │  │  Kafka  — domain events                     │
│  · rpc queues:  identity.rpc · training.rpc · activity.rpc · analysis.rpc   │  │  topics: identity · training · activity ·   │
│    (@MessagePattern, reply queue, timeout 5 s)                              │  │          analysis                            │
│  · work queues: import.parse-file · activity.compute-load ·                 │  │  facts, append-only, replayable, keyed by   │
│    session.compare · session.mark-missed  (ack/retry/DLQ)                   │  │  athlete_id; projections rebuilt from here  │
└───┬──────────────┬──────────────┬──────────────┬────────────────────────────┘  └───▲──────────▲──────────▲──────────▲───────┘
    │              │              │              │  ▲ services also send() each other   │          │          │          │
    ▼              ▼              ▼              ▼  │ (training → identity.membersOn)    │ produce  │          │          │ consume
┌──────────────┐  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐              │          │          │          │
│ identity-svc │  │ training-svc  │  │ activity-svc  │  │ analysis-svc  │              │          │          │          │
│ NestJS       │  │ NestJS        │  │ NestJS        │  │ NestJS        │──────────────┘          │          │          │
│ MySQL        │  │ MySQL         │  │ MySQL         │  │ MySQL         │─────────────────────────┘          │          │
│ identity_db  │  │ training_db   │  │ activity_db   │  │ analysis_db   │────────────────────────────────────┘          │
│ assessoria   │  │ prescription  │  │ activity      │  │ comparison    │───────────────────────────────────────────────┘
│ coach        │  │ workout_block │  │ activity_stream│ │ engine        │
│ athlete      │  │ assignment    │  │ activity_lap  │  │ dashboard     │
│ training_group│ │ session       │  │ file import   │  │ read models   │
│ membership   │  │ tolerance     │  │ load calc     │  │               │
│ credentials  │  │ mark-missed   │  │               │  │               │
└──────────────┘  └───────────────┘  └───────────────┘  └───────────────┘
```

Reading the diagram: the app and the watch only ever see the gateway; the gateway only ever sees the brokers; services answer RabbitMQ and publish/consume Kafka. Nothing calls a service by hostname.

### Who owns what (service boundaries)

| Service | MySQL schema | Tables (from MODEL.md) | Owns these use cases |
|---|---|---|---|
| **api-gateway** | `gateway_db` | `refresh_token`, `invite`, `idempotency_key` | UC01 (token issuing), route composition |
| **identity-svc** | `identity_db` | `assessoria`, `coach`, `athlete`, `athlete_credential`, `training_group`, `group_membership` | UC01 (credential check), UC02–UC06, UC24 |
| **training-svc** | `training_db` | `workout_prescription`, `workout_block`, `prescription_assignment`, `session`, `tolerance_setting` | UC13–UC16, UC22, UC23 |
| **activity-svc** | `activity_db` | `activity`, `activity_stream`, `activity_lap`, `import_job` | UC09 (file import), UC11 (sync), UC18 (load), manual entry |
| **analysis-svc** | `analysis_db` | `session_result` (adherence, confidence, comparison JSON), `dashboard_week` (projection) | UC17, UC19 (data), UC20, UC21 |

`provider_connection` is **dropped** from the MVP schema. `activity.source` stays: `'file_upload' | 'manual'`, with `'garmin_ciq'` reserved for Block 9 (and `'garmin'` if the Activity API reopens).

**Two consequences of splitting the schema that the monolith design didn't have:**

1. **The group-dashboard query of MODEL.md §9 joins three services** (`session` × `group_membership` × `activity`). It cannot be one SQL statement anymore. analysis-svc keeps a **projection** (`dashboard_week`, and a local copy of `membership_window` and `session_summary`) built from Kafka events, and the §9 query runs against that projection. The date-window rule (invariant 4) is applied when projecting, and the projection is rebuildable by replaying Kafka from offset 0.
2. **Personal data stays in identity-svc only** (invariant 2 gets stronger, not weaker). training-svc, activity-svc and analysis-svc store `athlete_id` and nothing else about the person. The gateway resolves names for the screen (`send('identity.athletes.batch', { ids })`). Anonymizing an athlete is still one `UPDATE` in one place.

### Messaging contract

| Channel | Use | Why this one |
|---|---|---|
| **RabbitMQ — request/response** (`ClientProxy.send()` → `@MessagePattern`) | Everything the gateway needs an answer for: reads (`identity.athletes.list`, `analysis.dashboard.group`) and writes the screen must confirm (`identity.athletes.create`, `training.prescriptions.assign`). Also the few service-to-service reads (`training` → `identity.groups.membersOn`) | one queue per service (`<service>.rpc`), auto reply queue, 5 s timeout → gateway returns `504`; services never exposed over HTTP |
| **RabbitMQ — commands** (`ClientProxy.emit()` → `@EventPattern`, durable work queues) | Things to do that don't need an answer now: `import.parse-file`, `activity.compute-load`, `session.compare`, `session.mark-missed`. Gateway answers the app `202 Accepted` | one worker consumes, ack/nack, retry with backoff, dead-letter queue |
| **Kafka — domain events** | Things that happened: `identity.athlete.inactivated`, `identity.membership.changed`, `identity.athlete.parameters_changed`, `training.session.planned/completed/missed`, `training.prescription.updated`, `activity.imported`, `activity.load_computed`, `analysis.session.compared` | many consumers (services **and the gateway**, which turns them into push notifications), retained & replayable (rebuild read models, audit), ordered per key (`athlete_id`) |

Rules: messages carry `assessoria_id` and `athlete_id`, never names. Every event and command has an `id` (UUID) and consumers are idempotent (upsert by id). NestJS `@nestjs/microservices`: each service registers both `Transport.RMQ` (its rpc queue + the work queues it owns) and `Transport.KAFKA` (consumer group per service). Message pattern naming: `<service>.<resource>.<action>`.

Why not everything on one broker: Kafka has no reply semantics, so it can't carry the gateway's reads; RabbitMQ doesn't retain, so it can't rebuild a projection. Each does the job it's built for.

### MySQL translation of MODEL.md (Postgres → MySQL 8)

| Postgres in MODEL.md | MySQL 8 |
|---|---|
| `uuid` | `CHAR(36)` (or `BINARY(16)` if size matters later); generated client-side |
| `citext` | `VARCHAR(255)` with `utf8mb4_0900_ai_ci` collation (case-insensitive by default) |
| `timestamptz` | `DATETIME(6)`, always UTC; app converts to assessoria timezone |
| `jsonb` | `JSON` |
| `bytea` | `LONGBLOB` (streams), `VARBINARY` (small) |
| `numeric(8,2)` | `DECIMAL(8,2)` |
| `text[]` (`keys`, `scopes`) | `JSON` array |
| `check (num_nonnulls(athlete_id, group_id) = 1)` | `CHECK ((athlete_id IS NULL) <> (group_id IS NULL))` |
| `count(*) filter (where ...)` | `SUM(status = 'missed')` |
| `date_trunc('week', d)` | `STR_TO_DATE(CONCAT(YEARWEEK(d, 1), ' Monday'), '%X%V %W')` — or store `week_start` in the projection (preferred) |
| `insert ... on conflict do nothing` | `INSERT IGNORE` / `INSERT ... ON DUPLICATE KEY UPDATE id = id` |
| `select ... for update` | same |

ORM: TypeORM or Prisma — pick one for all services. Migrations per service, per schema.

### Mobile: local database choice

The app needs a local DB for two things: **read cache** (open the app on the track with no signal and see today's workout) and an **outbox** (manual activity, feedback written offline, file uploads queued).

| Option | Verdict |
|---|---|
| **SQLite via `expo-sqlite` (Expo) or `op-sqlite` (bare RN) + Drizzle ORM** — **recommended** | Same TypeScript/SQL mindset as the NestJS side, schema shared through `packages/shared`, no vendor runtime, sync is a plain cursor pull + outbox push we control |
| WatermelonDB | Good if you want a full offline-first sync protocol out of the box; more opinionated, its own model layer |
| Realm | **Do not** — MongoDB deprecated the React Native Realm SDK (2024) |
| MMKV / AsyncStorage | Key-value only; use MMKV for small prefs, not as the DB |
| `react-native-keychain` / `expo-secure-store` | For JWT refresh token — never in SQLite |

Sync design (simple, sufficient for MVP): every entity has `updated_at`; app pulls `GET /sync?since=<cursor>` per screen scope; pushes outbox rows with client-generated UUIDs so retries are idempotent (the schema already uses UUID PKs).

### Invariants enforced in code review

1. `assessoria_id` is on every table in every schema and in every query. Tenant comes from the token (gateway) and is propagated in internal calls/events, never read from the request body.
2. Personal data lives **only** in `identity_db.athlete` / `identity_db.coach`. Not in other services, not in events, not in logs, not in the mobile cache beyond the athlete's own profile and the coach's athlete list.
3. `adherence_pct` null ≠ zero. No `COALESCE(adherence_pct, 0)` anywhere, including the projection.
4. Membership window `joined_at <= d AND (left_at IS NULL OR left_at > d)` is applied in `identity-svc.membersOn()` and in the analysis projection — nowhere else re-implements it.
5. Sessions are created at **assignment** (training-svc), never when an activity arrives.
6. Every consumer of Kafka/RabbitMQ is idempotent.

---

## Block 0 — Foundations

- [ ] Monorepo: `apps/mobile`, `apps/api-gateway`, `apps/identity-svc`, `apps/training-svc`, `apps/activity-svc`, `apps/analysis-svc`, `packages/shared` (DTOs, enums, zod/class-validator schemas, event & command types).
- [ ] `docker-compose`: MySQL 8 (one server, 5 schemas), Kafka (KRaft, no ZooKeeper), RabbitMQ (management UI on).
- [ ] NestJS skeleton per service: config module, health endpoint, Kafka producer/consumer module, RabbitMQ client/worker module, MySQL connection, migrations.
- [ ] Shared enums: `CoachRole`, `SessionStatus`, `Confidence`, `BlockType`, `WorkoutType`, `Source ('file_upload'|'manual')`, `LoadFormula`.
- [ ] Gateway: request id, structured logging (**no personal data**), error envelope `{ code, message, details? }`, tenant middleware, one RabbitMQ `ClientProxy` per service (`identity.rpc`, `training.rpc`, `activity.rpc`, `analysis.rpc`) that injects `meta: { assessoria_id, request_id, actor }` into every `send()`/`emit()`, Kafka consumer for push notifications.
- [ ] Mobile: RN app, navigation skeleton, SQLite + Drizzle set up, secure storage, API client with token refresh interceptor, outbox table.
- [ ] Seed: one assessoria, one admin coach, one coach, 3 athletes, 1 group.

**Done when:** `docker compose up` brings everything up, every service answers `/health`, the app boots and hits the gateway.

---

## Block 1 — Authentication & login (UC01)

**Owner:** api-gateway (tokens) + identity-svc (credentials).

> ⚠️ **Gap in MODEL.md:** UC01 says the Athlete authenticates, but `athlete` has **no `password_hash`** and `email` is nullable. Options: **(A)** add `password_hash` to `athlete`; **(B)** invite + set password on first login (adds `invite` table in gateway_db). **Recommendation: B**, which degrades to A after first login. Store credentials in `identity_db.athlete_credential (athlete_id PK, password_hash, updated_at)` to keep `athlete` as the pure personal-data row.

### Screens (mobile)
| Screen | Who | Notes |
|---|---|---|
| Splash / session restore | both | refresh token from secure storage → silent renew → role router |
| Login | both | email + password; server returns which role the user is |
| Accept invite / set password | athlete | deep link `pulsar://invite/<token>` |
| Forgot / reset password | both | email → deep link |
| Role router | both | coach → Coach Home, athlete → Athlete Home |

### Server actions (gateway)
| Method | Route | → service | Rules |
|---|---|---|---|
| POST | `/auth/login` | identity-svc `POST /internal/credentials/verify` | try coach then athlete; generic error (no user enumeration); inactive users rejected |
| POST | `/auth/refresh` | gateway_db | rotate refresh token; reject if user inactivated since issue (check `identity.athlete.inactivated` / `identity.coach.inactivated` events kept in a gateway denylist) |
| POST | `/auth/logout` | gateway_db | revoke refresh token |
| POST | `/auth/invite/accept` | identity-svc set credential | single-use, expiring token |
| POST | `/auth/password/forgot` · `/reset` | | |
| GET | `/me` | identity-svc | type, role, assessoria name |

### Authorization
- JWT claims: `sub`, `type: 'coach'|'athlete'`, `assessoriaId`, `role` (coach only). Access token short (15 min), refresh long, rotated.
- Gateway guards: `requireCoach()`, `requireAdmin()`, `requireAthlete()`; every downstream call carries `assessoriaId`.
- `Coach.canPrescribeFor(athlete)` = same `assessoria_id` (D5). One function in identity-svc; gateway trusts it.

**Done when:** coach and athlete log in on a device, survive an app restart, and a coach of assessoria X gets 404 for any athlete of assessoria Y.

---

## Block 2 — Coach: people management (UC02, UC03, UC04, UC24)

**Owner:** identity-svc. **Events out:** `identity.athlete.created`, `identity.athlete.inactivated`, `identity.athlete.parameters_changed`, `identity.coach.inactivated`.

### Screens
| Screen | Notes |
|---|---|
| Coach Home | tabs: Athletes · Groups · Workouts · Dashboard · Settings |
| Athlete list | active by default, toggle inactive; badge **"threshold not set"** when `threshold_pace_spk` is null. Cached locally |
| Athlete create (UC03) | name (req), email, phone, birth date; sends invite if email present |
| Athlete detail | header · training parameters · groups (current + history) · recent sessions (Block 8) |
| Athlete parameters (UC24) | `threshold_pace_spk` typed as `m:ss /km`, stored as seconds; `max_heart_rate`, `resting_heart_rate`. Presented as a **required onboarding step** |
| Inactivate athlete (UC04) | confirm; "history kept, athlete can no longer log in" |
| Coach list / create / inactivate (UC02) | **admin only**; role `coach | admin` |
| Assessoria settings | admin only: name, timezone, tolerances (Block 6) |

### Server actions (gateway → identity-svc)
| Method | Route | Guard | Notes |
|---|---|---|---|
| GET | `/athletes?active=` | coach | |
| POST | `/athletes` | coach | UC03; `UNIQUE (assessoria_id, email)`; creates invite |
| GET | `/athletes/:id` | coach | + current memberships |
| PATCH | `/athletes/:id` | coach | personal fields |
| PUT | `/athletes/:id/parameters` | coach | UC24; pace 120–1200 s/km, HR 30–230; emits `parameters_changed`; **does not** recompute old loads |
| POST | `/athletes/:id/inactivate` | coach | `is_active=false`, `inactivated_at=now()`, closes open memberships (`left_at=today`), emits event → gateway revokes tokens, activity-svc cancels pending imports |
| POST | `/athletes/:id/reactivate` | coach | memberships **not** reopened |
| GET/POST/PATCH | `/coaches`, `POST /coaches/:id/inactivate` | admin | last admin cannot be inactivated |

**Rule:** `AthleteService.anonymize()` exists (one `UPDATE` on `identity_db.athlete`) even though no screen calls it in MVP.

**Done when:** coach registers an athlete, sets threshold, athlete accepts invite and logs in; inactivating blocks the login within one token refresh.

---

## Block 3 — Coach: groups (UC05, UC06)

**Owner:** identity-svc. **Events out:** `identity.membership.changed { athlete_id, group_id, joined_at, left_at }` (analysis-svc projects it).

### Screens
| Screen | Notes |
|---|---|
| Group list | active groups, member count as of today |
| Group create / rename / inactivate (UC05) | |
| Group detail | members as of a chosen date (default today); add / remove |
| Add athlete (UC06) | active athletes not currently in the group; `joined_at` defaults today, backdating allowed, future not |
| Remove | sets `left_at`; never deletes |
| Membership history (in Athlete detail) | timeline |

### Server actions (gateway → identity-svc)
| Method | Route | Notes |
|---|---|---|
| GET/POST/PATCH | `/groups` | |
| GET | `/groups/:id/members?on=YYYY-MM-DD` | `GroupService.membersOn(groupId, date)` — the one implementation of the window query; also exposed internally for training-svc (assignment) |
| POST | `/groups/:id/members` | `{ athleteId, joinedAt }`; reject if open membership exists |
| POST | `/groups/:id/members/:mid/leave` | `{ leftAt }`; `CHECK (left_at >= joined_at)` |

**Done when:** moving an athlete from Group A to B in April leaves March in Group A (unit test on `membersOn`, integration test on the analysis projection).

---

## Block 4 — Athlete: profile & activity entry (UC09 as file import, manual entry)

**Owner:** identity-svc (profile), activity-svc (import, manual). Strava connection removed.

### Screens
| Screen | Notes |
|---|---|
| Athlete Home | next planned workout, last activity, pending uploads in outbox. Works offline from cache |
| Profile | own data (phone editable; name/email read-only unless decided otherwise), training parameters **read-only**, groups |
| Import activity file (UC09) | pick `.fit` / `.gpx` / `.tcx` (single or the watch app's export `.zip`) from device files or share-sheet ("Open in Pulsar" from Garmin/Coros/Strava export); shows job progress. D9 (formats) — MVP: FIT + GPX + TCX |
| Manual activity | date/time, type, distance, duration, optional avg HR. `source='manual'`. Works offline → outbox |
| Activity list / detail | own; map allowed on own activities only |

### Server actions (gateway)
| Method | Route | → service | Notes |
|---|---|---|---|
| GET/PATCH | `/me/profile` | identity-svc | |
| POST | `/me/imports` (multipart) | activity-svc | creates `import_job`, stores file, publishes RabbitMQ `import.parse-file`. Coach may also call `POST /athletes/:id/imports` on the athlete's behalf (onboarding history) |
| GET | `/me/imports/:id` | activity-svc | total / imported / duplicates / failed |
| POST | `/me/activities` | activity-svc | manual; `Idempotency-Key` = client UUID |
| GET | `/me/activities`, `/me/activities/:id` | activity-svc | |

**Done when:** an athlete shares a `.fit` file from the watch app into Pulsar and sees it as an activity within a minute; a manual activity entered in airplane mode appears after reconnecting, once.

---

## Block 5 — Activities: sync pipeline (UC11, UC18)

**Owner:** activity-svc. **Queues in:** `import.parse-file`, `activity.compute-load`. **Events out:** `activity.imported`, `activity.load_computed`.

Two entry points, one sync function:

```
file import (UC09) ─┐
                    ├─▶ ActivityService.sync(athleteId, source, sourceRef, parsed)
manual entry       ─┘        │
                             ├─ dedupe: INSERT IGNORE on UNIQUE (source, source_ref)
                             │    source_ref = hash of file content (file_upload) | client UUID (manual)
                             ├─ persist activity, activity_stream (compressed), activity_lap
                             ├─ has_usable_laps (FIT usually yes, GPX from phone usually no)
                             ├─ RabbitMQ → activity.compute-load (UC18)
                             └─ Kafka → activity.imported { activity_id, athlete_id, started_at, distance, ... }
                                          │
                                          ▼ consumed by training-svc
                                training-svc links the session:
                                  · planned/missed session for same athlete + date → attach, status=completed,
                                    RabbitMQ → session.compare (UC17)
                                  · none → create session status=unplanned
                                  · more than one prescription that day → closest by distance, flag ambiguity
```

> **Late upload flips `missed` → `completed`.** Without Strava's webhook there is no reconciliation job; the athlete uploads when they get around to it. training-svc must accept an activity for a session already marked `missed` and restore it to `completed`. This replaces UC25.

### Coach screens
| Screen | Notes |
|---|---|
| Athlete activity list / detail | via `GET /athletes/:id/activities`; **no map** (privacy of `latlng`); load or "threshold not set" |

### Jobs
| Job | Queue | Rules |
|---|---|---|
| `parseFile` | `import.parse-file` | unzip if needed; one `sync()` per file; per-file result recorded in `import_job` |
| `computeLoad` (UC18) | `activity.compute-load` | reads athlete parameters via `send('identity.athletes.parameters', { id })`; `pace_v1` if threshold set else `training_load=NULL`; stores `load_formula`, `load_inputs` (with the threshold used). `hr_v1` is post-pilot |

**Rule:** `parameters_changed` never rewrites past loads. Recalculation is an explicit admin job, out of MVP.

**Done when:** the same file uploaded twice produces one activity; a manual activity gets a load; a FIT with laps sets `has_usable_laps=true`.

---

## Block 6 — Coach: prescription, assignment, sessions (UC13, UC14, UC15, UC16, UC23)

**Owner:** training-svc. **Events out:** `training.session.planned`, `training.session.missed`, `training.session.completed`, `training.prescription.updated`. **Queue in:** `session.mark-missed` (cron), `session.link-activity` (from `activity.imported`).

### Screens
| Screen | Notes |
|---|---|
| Workout list | by scheduled date; filter group/athlete |
| Workout form (UC13/UC15) | flat form from PRESCRIPTION.md §3: header (name, type, date, notes, **assign to athlete or group**), warm-up, main set (required), cool-down; `m:ss` typed, seconds stored; free-text toggle (`is_structured=false`). Draft saved in local DB |
| Workout detail | blocks + assignments + per-athlete status (planned / completed / missed) |
| Assign (UC14) | one athlete or one group; multiple assignments allowed |
| Tolerances (UC23) | admin; per `workout_type`: pace faster %, pace slower %, distance % |
| Athlete: My workouts | upcoming + past, cached for offline |

### Server actions (gateway → training-svc)
| Method | Route | Guard | Notes |
|---|---|---|---|
| POST | `/prescriptions` | coach | validation PRESCRIPTION.md §4; prescription + blocks in one transaction |
| PATCH | `/prescriptions/:id` | coach | date change moves `planned` sessions; never touches `completed` results |
| GET | `/prescriptions?from&to&groupId&athleteId` | coach | |
| GET | `/me/sessions?from&to` | athlete | |
| POST | `/prescriptions/:id/assign` | coach | `{ athleteId }` xor `{ groupId }`. Group → identity-svc `membersOn(groupId, scheduled_date)`. One `session` per athlete, `status='planned'`; `UNIQUE (athlete_id, prescription_id)` makes it idempotent |
| DELETE | `/prescriptions/:id/assignments/:aid` | coach | only if all sessions still `planned` |
| GET/PUT | `/settings/tolerances` | admin | |

### Scheduler job (UC16)
`markMissed`: NestJS `@Cron` in training-svc, once per hour, per assessoria timezone: `UPDATE session SET status='missed' WHERE status='planned' AND scheduled_date < <local today>`. Emits `training.session.missed`. Late uploads can still flip it back (Block 5).

**Done when:** assigning to Group A creates one planned session per member as of that date; a member who joins tomorrow gets none; sessions become `missed` the next local day and go back to `completed` when the file arrives late.

---

## Block 7 — Comparison & coach review (UC17, UC19, UC22)

**Owner:** analysis-svc (engine, results), training-svc (`coach_feedback`). **Queue in:** `session.compare`. **Events out:** `analysis.session.compared { session_id, adherence_pct, confidence }`.

MVP builds **Path A only** (usable laps). Path B waits for pilot numbers — with FIT files from watches the share of Path A should be high; GPX from phones will be Path B.

### Screens
| Screen | Who | Notes |
|---|---|---|
| Session detail | coach, athlete | prescribed vs achieved per repetition, `adherence_pct`, `confidence` badge; `low`/`not_compared` → **"could not compare"**, never 0 % |
| Review queue (UC19) | coach | `confidence='low'` or ambiguity flag; relink to another prescription or mark reviewed |
| Feedback (UC22) | coach writes, athlete reads | offline-capable via outbox |

### Server actions
| Method | Route | → service |
|---|---|---|
| GET | `/sessions/:id` | training-svc + analysis-svc (gateway composes) |
| GET | `/sessions/review-queue` | analysis-svc |
| POST | `/sessions/:id/relink` `{ prescriptionId }` | training-svc → re-enqueue `session.compare` |
| PUT | `/sessions/:id/feedback` | training-svc |

### Service
`ComparisonEngine.compare(cmd)`: fetch blocks (training-svc), streams + laps (activity-svc), tolerance (training-svc) → `lapsAreUsable` → Path A slices `stream.slice(lap.startIndex, lap.endIndex)` → judge per repetition, copy `tolerance_used` → confidence → persist `session_result`, emit event. `is_structured=false` → `not_compared`, `adherence_pct=NULL`.

---

## Block 8 — Dashboards (UC20, UC21)

**Owner:** analysis-svc, read models projected from Kafka:
- `membership_window(assessoria_id, group_id, athlete_id, joined_at, left_at)` ← `identity.membership.changed`
- `session_summary(session_id, assessoria_id, athlete_id, scheduled_date, status, activity_id, training_load, adherence_pct, confidence)` ← `training.session.*`, `activity.load_computed`, `analysis.session.compared`
- `dashboard_week` optional materialization if the query gets slow

### Screens
| Screen | Who | Notes |
|---|---|---|
| Individual dashboard (UC20) | athlete (own), coach (any) | week view: planned / completed / missed, weekly load, adherence (nulls excluded, "n of m compared") |
| Group dashboard (UC21) | coach | group + period → per week: athletes, avg load, avg adherence, misses. Query is MODEL.md §9 rewritten over `session_summary` × `membership_window` in MySQL (`AVG` ignores NULL, `SUM(status='missed')`). No maps. Empty-state: "needs a few weeks of data" |

### Server actions
| Method | Route |
|---|---|
| GET | `/athletes/:id/dashboard?from&to` (coach) · `/me/dashboard` (athlete) |
| GET | `/groups/:id/dashboard?from&to` (coach) |

---

---

## Block 9 — Watch App (Garmin Connect IQ) — post-pilot, conditional

**Goal:** capture the run on the watch and deliver it to Pulsar without the athlete exporting a file. Built only if the pilot shows that FIT upload (Block 4) is not enough.
**Owner:** new app `apps/watch-ciq` (Monkey C) + api-gateway routes + activity-svc (`source='garmin_ciq'`). No new microservice.

### Why a data field, not a device app
A Connect IQ app **cannot read the FIT file** of a run recorded by the native Run app. It can only see data **while** a run is being recorded. So Pulsar ships a **data field** the athlete adds to their normal Run screen; it samples pace/HR/distance/laps during the run and posts them to the gateway. The native Run app keeps recording as usual — nothing changes for the athlete except one extra field on screen.

### Flow — watch → API layer → services → mobile app (one-way from the watch)

The watch only **sends**. It never receives data from the server — no prescription on the wrist, no results, no tokens pushed down. The HTTP response to its `makeWebRequest` is a bare acknowledgment (`204`, or `401` if the watch is not paired); the field uses it only to decide whether to drop the chunk from its buffer. Everything processed comes back to the **mobile app**.

```
┌── Garmin watch ───────────────────────┐
│  Pulsar data field (Monkey C)         │
│  · self-generated device_id (once)    │
│  · Activity.getActivityInfo() @1 Hz   │
│  · onTimerLap → lap boundary          │
│  · buffer in Application.Storage      │
│  · every ~5 min + on stop:            │
│    Communications.makeWebRequest      │
└──────────────┬────────────────────────┘
               │ HTTPS via phone's Garmin Connect app (BLE bridge) or Wi-Fi
               │ POST /ciq/runs/:runId/chunks   X-Device-Id
               │ ◀ 204 / 401 only (ack, no body)
               ▼
┌── api-gateway ────────────────────────┐        ┌── activity-svc ─────────────────────┐
│ /ciq/runs/*                           │ ─────▶ │ ciq_run (chunks) → on finish:        │
│ device_id → athlete_id (paired in app)│        │ assemble → sync(source='garmin_ciq') │
└───────────────────────────────────────┘        │ → activity.compute-load              │
                                                 └──────────────┬──────────────────────┘
                                                                │ Kafka activity.imported
                                                                ▼
                                                  training-svc links session → completed
                                                  RabbitMQ session.compare → analysis-svc
                                                                │ Kafka analysis.session.compared
                                                                ▼
                                           ┌── api-gateway: push / sync cursor ──────────┐
                                           │ FCM/APNs "Run received · 6/6 reps · 92 %"    │
                                           │ GET /sync?since=  → activity + session       │
                                           └──────────────────────┬───────────────────────┘
                                                                  │ processed data → app
                                                                  ▼
                              ┌── Mobile app (React Native) ─────────────────────────────┐
                              │ Athlete: Home shows the run, session detail with          │
                              │   adherence/confidence, activity detail                   │
                              │ Coach:  athlete list badge "new run", review queue if low │
                              └───────────────────────────────────────────────────────────┘
```

**What the server returns, and to whom**

| To | When | Payload |
|---|---|---|
| **Watch** | every `makeWebRequest` | **nothing** — `204 No Content` (accepted) or `401` (device not paired). No body, no ids, no prescription. The watch is a sender only |
| **Mobile app** (push notification) | after `analysis.session.compared` (or after `activity.imported` if not comparable) | `{ type: 'run_received', session_id, activity_id, adherence_pct?, confidence }` |
| **Mobile app** (sync pull) | next `GET /sync?since=` | new `activity` (`source='garmin_ciq'`), updated `session` (`completed`), `session_result` |
| **Mobile app** (device status) | `GET /me/ciq/device` | `{ device_id, paired_at, last_seen_at, last_run_at }` — how the athlete knows the watch is talking to the server |

### Screens
| Where | Screen | Notes |
|---|---|---|
| Watch | Data field | one line: `Pulsar ✓` (last chunk acked) / `!` (unsent chunks or 401 not paired). Nothing from the server is ever displayed |
| Watch | Device code | settings/first launch: shows the watch's own code, e.g. `PLS-483921` (derived from its self-generated `device_id`). The athlete reads it off the wrist and types it into the app — the watch does not poll or receive anything |
| Mobile (athlete) | Connect watch | types the code → `POST /me/ciq/device`; shows device status (`last_seen_at`, `last_run_at`), "unpair" |
| Mobile (athlete) | Home | banner "Run received from watch" → session detail (adherence/confidence) |
| Mobile (coach) | Athlete list / review queue | unchanged; runs from the watch look like any other activity, `source='garmin_ciq'` visible in detail |

### Server actions (gateway)
| Method | Route | Auth | → | Notes |
|---|---|---|---|---|
| POST | `/me/ciq/device` | athlete JWT (app) | gateway_db | `{ code }` → binds `device_id` to `athlete_id`, `assessoria_id`; one active device per athlete (re-pair replaces) |
| GET | `/me/ciq/device` | athlete JWT (app) | gateway_db | status: `paired_at`, `last_seen_at`, `last_run_at` |
| DELETE | `/me/ciq/device` | athlete JWT (app) | gateway_db | unpair; subsequent watch posts get `401` |
| POST | `/ciq/runs` | `X-Device-Id` (watch) | activity-svc | `{ run_id, started_at }` — `run_id` is generated **on the watch** so it never needs an answer; `204` |
| POST | `/ciq/runs/:id/chunks` | `X-Device-Id` (watch) | activity-svc | `{ seq, samples: [[t, dist_m, pace_spk, hr]...], laps: [[start_t, end_t]...] }`; idempotent by `(run_id, seq)`; `204` |
| POST | `/ciq/runs/:id/finish` | `X-Device-Id` (watch) | activity-svc | assembles chunks → `sync()`; `source_ref = device_id + started_at`; `204` — the resulting `activity_id` goes to the app, not the watch |

Watch → server calls are the only direction. Unpaired or unknown `device_id` → `401`, and the chunk is kept on the watch for retry. Every other row in this table is app ↔ server.

### activity-svc additions
- Table `ciq_run (id, assessoria_id, athlete_id, device_id, started_at, status, last_seq)` and `ciq_run_chunk (run_id, seq, payload JSON)`.
- `finish` builds an in-memory stream (`time`, `distance`, `pace`, `heartrate`; **no `latlng` in v1** — the field doesn't send GPS, which also sidesteps the privacy rule) and laps with `start_index/end_index`, then calls the same `sync()` as file upload. `has_usable_laps = true` when the athlete pressed lap on the watch.
- Dedupe against a later FIT upload of the **same run**: if a `file_upload` arrives with `started_at` within ±2 min of a `garmin_ciq` activity for the same athlete, keep the FIT (richer), relink the session, mark the CIQ activity `superseded`. Not a schema change — a `superseded_by` column on `activity`.

### Rules & constraints
- **Garmin Connect on the phone is required and cannot be replaced by the Pulsar app.** The watch ↔ phone BLE protocol is proprietary; `makeWebRequest` and the Connect IQ Mobile SDK both route through Garmin Connect. This costs the athlete nothing — it is already installed, since a Garmin watch cannot be set up without it — and they never open it for Pulsar. (Theoretical alternative: `Toybox.BluetoothLowEnergy` with the RN app as a GATT peripheral. Rejected: large native/BLE effort to remove a dependency the athlete already has.)
- Payload discipline: chunks ≤ ~4 KB; data fields on older watches have tens of KB total. Sample at 1 Hz but post every 5 min or 300 samples, whichever first.
- `makeWebRequest` from a data field is limited to about one call per 5 minutes; the `finish` call happens on activity stop.
- If the phone isn't reachable, chunks stay in `Application.Storage` and are retried on the next run start; after 24 h the watch drops them and shows `!` (the athlete can still upload the FIT).
- **One-way:** the watch never consumes a response body. All ids it needs (`device_id`, `run_id`, `seq`) are generated on the watch. If someday something must reach the wrist (e.g. today's target pace), that is a new decision, not an extension of this block.
- `device_id` is per watch, revocable from the app, and carries no personal data; the binding to the athlete lives only in `gateway_db`.
- Monkey C app type is fixed in `manifest.xml` — `datafield`. Changing later means a new store listing.

### Dependencies & sizing
- Depends on Blocks 4/5 (`sync()`), 6 (session link), 7 (compare) and push notifications (currently post-MVP — this block is the first real reason to build them).
- Size: **L** — Monkey C + simulator, a physical watch for testing, Connect IQ Store review, plus the gateway/activity-svc work above.

**Done when:** an athlete pairs the watch once, runs a 6×800 with the field on screen and laps pressed, and within a minute of stopping gets a push on the phone showing 6/6 reps and an adherence %, with the coach seeing the same session as `completed` — no file touched.

---
---

# Service work blocks (S1–S4)

Feature blocks 0–8 are the delivery order and cover mobile, gateway and messaging. The four blocks below are the **same work seen per microservice**: one block = one service = one backlog for whoever owns it. Mobile and api-gateway are not repeated here — their work stays inside the feature blocks.

Each service block: schema · request/response message patterns (RabbitMQ rpc, called by the gateway or another service) · events produced / consumed (Kafka) · commands consumed / produced (RabbitMQ work queues) · cron · rules · tests · done-when · which feature blocks it unblocks.

Conventions for all services:
- Services have **no HTTP endpoints** except `/health`. Every row in a "Message patterns" table is a `@MessagePattern('<service>.<resource>.<action>')` handler on the service's `<service>.rpc` queue. The `Method | Route` columns are kept as a readable shorthand: `GET /internal/athletes/:id` means pattern `identity.athletes.get` with payload `{ id }`; `POST /internal/athletes` means `identity.athletes.create` with the body as payload; and so on.
- Every message payload carries `meta: { assessoria_id, request_id, actor }` (`actor` = `coach:<id>` | `athlete:<id>` | `system`). Handlers reject a missing `assessoria_id`.
- Every table has `id CHAR(36)`, `assessoria_id CHAR(36)`, `created_at DATETIME(6)`, `updated_at DATETIME(6)`.
- Every event: `{ id, type, occurred_at, assessoria_id, payload }`, Kafka key = `athlete_id` when there is one, else `assessoria_id`.
- Every consumer is idempotent: processed event ids are stored in `<service>_db.processed_event (event_id PK, processed_at)`.

---

## S1 — identity-svc

**Purpose:** the only place that knows who people are. Tenant, coaches, athletes, credentials, groups, membership history.

### Schema (`identity_db`)
| Table | Notes |
|---|---|
| `assessoria` | + `timezone VARCHAR(64) NOT NULL DEFAULT 'America/Sao_Paulo'` (needed by training-svc mark-missed) |
| `coach` | `email` `utf8mb4_0900_ai_ci`, `UNIQUE (assessoria_id, email)`, `role ENUM('coach','admin')` |
| `athlete` | personal data **only here**; `threshold_pace_spk`, `max_heart_rate`, `resting_heart_rate`; `is_active`, `inactivated_at` |
| `athlete_credential` | `athlete_id PK FK`, `password_hash`, `updated_at` — separate so `athlete` stays the personal-data row |
| `training_group` | |
| `group_membership` | `joined_at DATE`, `left_at DATE NULL`, `CHECK (left_at IS NULL OR left_at >= joined_at)`, indexes `(group_id, joined_at, left_at)`, `(athlete_id)` |
| `processed_event` | |

### Message patterns (RabbitMQ request/response)
| Method | Route | Used by | Notes |
|---|---|---|---|
| POST | `/internal/credentials/verify` | gateway (UC01) | `{ email, password }` → `{ type, id, assessoriaId, role }` or 401; checks coach then athlete; inactive → 401 |
| PUT | `/internal/athletes/:id/credential` | gateway (invite accept, reset) | |
| GET | `/internal/me/:type/:id` | gateway | |
| GET/POST/PATCH | `/internal/athletes` `[/:id]` | gateway (Block 2) | list filters `active`, pagination |
| GET | `/internal/athletes/batch?ids=` | gateway | id → `{ name }` for screen enrichment; **the only way other data reaches a screen with a name on it** |
| PUT | `/internal/athletes/:id/parameters` | gateway (UC24) | validation ranges; emits `parameters_changed` |
| GET | `/internal/athletes/:id/parameters` | activity-svc (compute load) | `{ threshold_pace_spk, max_heart_rate, resting_heart_rate }` |
| POST | `/internal/athletes/:id/inactivate` · `/reactivate` | gateway (UC04) | inactivate closes open memberships in the same transaction |
| POST | `/internal/athletes/:id/anonymize` | nobody in MVP | exists; one `UPDATE` |
| GET/POST/PATCH | `/internal/coaches` `[/:id]`, `POST .../inactivate` | gateway (UC02) | last admin guard |
| GET/POST/PATCH | `/internal/groups` `[/:id]` | gateway (UC05) | inactivate closes open memberships |
| GET | `/internal/groups/:id/members?on=YYYY-MM-DD` | gateway, **training-svc (assign)** | `membersOn()` — the one implementation of the date window |
| POST | `/internal/groups/:id/members` · `POST .../members/:mid/leave` | gateway (UC06) | |
| GET | `/internal/athletes/:id/memberships` | gateway (history timeline) | |
| GET | `/internal/assessorias/:id` | gateway, training-svc (timezone) | |

### Events produced (Kafka topic `identity`)
| Type | Payload | Consumers |
|---|---|---|
| `identity.athlete.created` | `{ athlete_id }` | — (future) |
| `identity.athlete.inactivated` | `{ athlete_id, inactivated_at }` | gateway (token denylist), activity-svc (cancel pending imports), training-svc (no new sessions) |
| `identity.athlete.reactivated` | `{ athlete_id }` | gateway |
| `identity.coach.inactivated` | `{ coach_id }` | gateway |
| `identity.athlete.parameters_changed` | `{ athlete_id, threshold_pace_spk, max_heart_rate, resting_heart_rate }` | — (activity-svc must **not** recompute on this; logged only) |
| `identity.membership.changed` | `{ membership_id, athlete_id, group_id, joined_at, left_at }` | analysis-svc (projection) |

### Events consumed
None.

### Rules
- No other service may store `name`, `email`, `phone`, `birth_date`. Reviewers reject PRs that add those columns elsewhere.
- Backdated `joined_at` allowed; future `joined_at` rejected; overlapping open membership in the same group rejected.
- Coach/athlete uniqueness is per tenant (`UNIQUE (assessoria_id, email)`), so the same email can exist in two assessorias — login must try both and return 409 "choose assessoria" if it matches more than one (or make email globally unique — **decide**).

### Tests
- `membersOn`: join in March, leave in April, ask for March 15 / April 15 / May 1.
- Inactivate athlete closes memberships and rejects login.
- Tenant leak test: every list endpoint with a foreign `x-assessoria-id` returns empty.

**Unblocks:** feature blocks 1, 2, 3, and (via `membersOn`) 6.

---

## S2 — training-svc

**Purpose:** what the coach planned and what state each athlete's session is in. Owns the prescription form, assignment fan-out, session lifecycle, tolerances, mark-missed.

### Schema (`training_db`)
| Table | Notes |
|---|---|
| `workout_prescription` | `workout_type ENUM(...)`, `scheduled_date DATE`, `is_structured`, `free_text` |
| `workout_block` | `block_type ENUM('warmup','main','cooldown')`, `UNIQUE (prescription_id, block_type, position)`, `ON DELETE CASCADE` |
| `prescription_assignment` | `CHECK ((athlete_id IS NULL) <> (group_id IS NULL))` |
| `session` | `status ENUM('planned','completed','missed','unplanned')`, `activity_id CHAR(36) NULL` (no FK — other schema), `coach_feedback`, `ambiguity_flag BOOL`, `UNIQUE (athlete_id, prescription_id)`, indexes `(athlete_id, scheduled_date)`, `(assessoria_id, scheduled_date)`, `(status)`. **`adherence_pct` / `confidence` / `comparison_result` are NOT here** — they live in analysis-svc |
| `tolerance_setting` | `UNIQUE (assessoria_id, workout_type)` |
| `processed_event` | |

### Message patterns (RabbitMQ request/response)
| Method | Route | Used by | Notes |
|---|---|---|---|
| POST | `/internal/prescriptions` | gateway (UC13) | PRESCRIPTION.md §4 validation; prescription + blocks in one transaction |
| PATCH | `/internal/prescriptions/:id` | gateway (UC15) | date change moves `planned` sessions; refuses block changes that would invalidate `completed` sessions? → **no**: allowed, but emits `prescription.updated` so analysis-svc can flag stale results |
| GET | `/internal/prescriptions?from&to&groupId&athleteId` `[/:id]` | gateway | |
| GET | `/internal/prescriptions/:id/blocks` | analysis-svc (compare) | |
| POST | `/internal/prescriptions/:id/assign` | gateway (UC14) | group → `send('identity.groups.membersOn', { groupId, on: scheduled_date })`; `INSERT IGNORE` one session per athlete |
| DELETE | `/internal/prescriptions/:id/assignments/:aid` | gateway | only if all its sessions are `planned` |
| GET | `/internal/sessions?athleteId&from&to` `[/:id]` | gateway (athlete "my workouts", session detail) | |
| POST | `/internal/sessions/:id/relink` | gateway (UC19) | `{ prescriptionId }` → re-enqueue `session.compare` |
| PUT | `/internal/sessions/:id/feedback` | gateway (UC22) | |
| GET/PUT | `/internal/tolerances` | gateway (UC23), analysis-svc (compare) | defaults from PRESCRIPTION.md §9 seeded per assessoria on first read |

### Events consumed
| Type | Action |
|---|---|
| `activity.imported` | **session linking** (UC11 step 7): find session for `athlete_id` + local date of `started_at` with status `planned` or `missed` → set `activity_id`, `status='completed'`, publish command `session.compare`. None → insert `unplanned` session. More than one candidate → closest by prescribed distance, `ambiguity_flag=true`. Emit `training.session.completed` / `training.session.unplanned` |
| `identity.athlete.inactivated` | delete future `planned` sessions of that athlete (they are not misses) |

### Events produced (topic `training`)
| Type | Payload | Consumers |
|---|---|---|
| `training.session.planned` | `{ session_id, athlete_id, prescription_id, scheduled_date }` | analysis-svc |
| `training.session.completed` | `{ session_id, athlete_id, activity_id, scheduled_date }` | analysis-svc |
| `training.session.unplanned` | `{ session_id, athlete_id, activity_id, scheduled_date }` | analysis-svc |
| `training.session.missed` | `{ session_id, athlete_id, scheduled_date }` | analysis-svc |
| `training.session.deleted` | `{ session_id }` | analysis-svc |
| `training.prescription.updated` | `{ prescription_id }` | analysis-svc |

### Commands produced (RabbitMQ)
| Queue | Payload | When |
|---|---|---|
| `session.compare` | `{ session_id, athlete_id, prescription_id, activity_id }` | on link, on relink |

### Cron
`markMissed` hourly: for each assessoria (timezone from identity-svc, cached), `UPDATE session SET status='missed' WHERE status='planned' AND scheduled_date < <today in tz>`; emit one `training.session.missed` per row.

### Rules
- Session is created at assignment, never on activity arrival — except `unplanned`.
- `missed` → `completed` on late upload is allowed; `completed` → anything is not (relink keeps `completed`).
- Free-text prescription (`is_structured=false`) still creates sessions; they just never get compared.

### Tests
- Assign to group with a member joining tomorrow → member gets no session.
- Activity arrives for a `missed` session → `completed` + compare command.
- Two prescriptions same day → closest by distance, flag set.
- Mark-missed respects timezone (23:30 BRT is still "today").

**Unblocks:** feature blocks 6, 7 (feedback, relink), and the session half of 5.

---

## S3 — activity-svc

**Purpose:** what the athlete actually did. File parsing, manual entry, dedupe, streams, laps, training load.

### Schema (`activity_db`)
| Table | Notes |
|---|---|
| `activity` | `source ENUM('file_upload','manual')`, `source_ref VARCHAR(128)` (file content SHA-256 | client UUID), `UNIQUE (source, source_ref)`, `training_load DECIMAL(8,2) NULL`, `load_formula ENUM('pace_v1','hr_v1') NULL`, `load_inputs JSON`, `has_hr_stream`, `has_usable_laps`, indexes `(athlete_id, started_at)`, `(assessoria_id, started_at)` |
| `activity_stream` | `activity_id PK`, `point_count`, `keys JSON`, `data LONGBLOB` (gzip of column-oriented JSON or protobuf — **decide**) |
| `activity_lap` | `UNIQUE (activity_id, lap_index)`, `start_index`, `end_index` into the stream |
| `import_job` | `athlete_id`, `requested_by` (`coach:<id>` | `athlete:<id>`), `file_key` (object storage path), `status ENUM('queued','parsing','done','failed')`, `total`, `imported`, `duplicates`, `failed`, `error` |
| `processed_event` | |

Object storage for uploaded files: local disk volume in dev, S3-compatible bucket in prod; retention 30 days after job done.

### Message patterns (RabbitMQ request/response)
| Method | Route | Used by | Notes |
|---|---|---|---|
| POST | `/internal/imports` (multipart or `{ file_key }`) | gateway (UC09) | creates `import_job`, publishes `import.parse-file` |
| GET | `/internal/imports/:id` | gateway | progress |
| POST | `/internal/activities` | gateway (manual, UC26) | `Idempotency-Key` header → `source_ref`; runs `sync()` inline (no streams) |
| GET | `/internal/activities?athleteId&from&to` | gateway | list, summary fields only |
| GET | `/internal/activities/:id?include=streams,laps&strip=latlng` | gateway (detail), analysis-svc (compare) | gateway passes `strip=latlng` when the caller is a coach |
| GET | `/internal/activities/batch?ids=` | gateway (session detail), analysis-svc | summaries |

### Commands consumed (RabbitMQ)
| Queue | Handler | Rules |
|---|---|---|
| `import.parse-file` | `parseFile(job)` | unzip; per file: detect format (FIT/GPX/TCX), parse → `sync()`; count imported/duplicate/failed; job never fails as a whole because one file is bad. Retry 3× with backoff, then DLQ |
| `activity.compute-load` | `computeLoad(activity_id)` | `send('identity.athletes.parameters', { id })`; no threshold → `training_load=NULL`, `load_formula=NULL`; else `pace_v1`; store `load_inputs = { threshold_pace_spk, moving_seconds, distance_meters }`; emit `activity.load_computed` |

### Commands produced
| Queue | When |
|---|---|
| `activity.compute-load` | after every successful `sync()` |

### Events produced (topic `activity`)
| Type | Payload | Consumers |
|---|---|---|
| `activity.imported` | `{ activity_id, athlete_id, source, started_at, activity_type, distance_meters, moving_seconds, has_usable_laps }` | training-svc (link session), analysis-svc |
| `activity.load_computed` | `{ activity_id, athlete_id, training_load, load_formula }` | analysis-svc |

### Events consumed
| Type | Action |
|---|---|
| `identity.athlete.inactivated` | cancel `queued` import jobs for that athlete |

### `sync()` contract (UC11)
1. `INSERT IGNORE` activity by `(source, source_ref)`; 0 rows → return `duplicate`.
2. Insert stream (compressed) and laps in the same transaction.
3. `has_usable_laps` = laps exist, ≥ 2, and lap distances are not all equal to total / n (auto-lap heuristic — refine in COMPARISON.md terms).
4. Commit, then publish command + event (outbox table or publish-after-commit; pick one and keep it).

### Rules
- Parsing runs only in the work-queue consumer, never inside an rpc handler (the gateway's `send()` would time out).
- `parameters_changed` does **not** trigger recompute. `POST /internal/admin/recompute-loads?athleteId` exists as a manual admin tool only.
- `latlng` never leaves the service unless the requester is the owning athlete.

### Tests
- Same FIT uploaded twice → one activity, job reports 1 duplicate.
- GPX without laps → `has_usable_laps=false`.
- Manual activity without threshold → load NULL, event still emitted.
- Load example from MODEL.md §5.1: 1 h at 5:00/km, threshold 4:30 → 81.00.

**Unblocks:** feature blocks 4, 5, and the activity half of 7.

---

## S4 — analysis-svc

**Purpose:** judge and aggregate. Comparison engine (Path A), session results, dashboard read models projected from Kafka.

### Schema (`analysis_db`)
| Table | Notes |
|---|---|
| `session_result` | `session_id PK`, `assessoria_id`, `athlete_id`, `adherence_pct DECIMAL(5,2) NULL`, `confidence ENUM('high','medium','low','not_compared')`, `tolerance_used JSON`, `comparison_result JSON` (repetitions), `path ENUM('A','B') NULL`, `stale BOOL` (prescription edited after compare), `reviewed_at` |
| `membership_window` | projection of `group_membership`: `membership_id PK`, `assessoria_id`, `group_id`, `athlete_id`, `joined_at`, `left_at`; indexes as MODEL.md |
| `session_summary` | projection: `session_id PK`, `assessoria_id`, `athlete_id`, `scheduled_date`, `week_start DATE`, `status`, `activity_id NULL`, `training_load NULL`, `adherence_pct NULL`, `confidence NULL`; indexes `(assessoria_id, week_start)`, `(athlete_id, scheduled_date)` |
| `processed_event` | |

### Message patterns (RabbitMQ request/response)
| Method | Route | Used by | Notes |
|---|---|---|---|
| GET | `/internal/sessions/:id/result` | gateway (session detail) | |
| GET | `/internal/sessions/review-queue?assessoriaId` | gateway (UC19) | `confidence='low'` or `stale=true` and `reviewed_at IS NULL` |
| POST | `/internal/sessions/:id/reviewed` | gateway (UC19) | |
| GET | `/internal/dashboard/athlete/:id?from&to` | gateway (UC20) | per week: planned / completed / missed, load sum, `AVG(adherence_pct)`, `compared_count / total_count` |
| GET | `/internal/dashboard/group/:id?from&to` | gateway (UC21) | MODEL.md §9 over `session_summary` × `membership_window` |
| GET | `/internal/pilot/path-split` | nobody in MVP UI | count of Path A vs B — the first-week pilot metric |
| POST | `/internal/admin/rebuild-projections` | ops | truncates projections, replays Kafka topics from offset 0 |

### Commands consumed (RabbitMQ)
| Queue | Handler |
|---|---|
| `session.compare` | `ComparisonEngine.compare(cmd)`: blocks ← `send('training.prescriptions.blocks')`, tolerances ← `send('training.tolerances.get')`, activity+streams+laps ← `send('activity.activities.get', { include: ['streams','laps'] })`; `is_structured=false` → `not_compared`; `lapsAreUsable` → Path A (slice by lap index) else `not_compared` in MVP (Path B later); judge per repetition; confidence; `low` → `adherence_pct=NULL`; upsert `session_result`; emit `analysis.session.compared` |

### Events consumed → projections
| Type | Projection update |
|---|---|
| `identity.membership.changed` | upsert `membership_window` |
| `training.session.planned/completed/unplanned/missed` | upsert `session_summary` (`status`, `activity_id`, `week_start = Monday of scheduled_date`) |
| `training.session.deleted` | delete from `session_summary`, `session_result` |
| `training.prescription.updated` | `session_result.stale=true` for its completed sessions |
| `activity.load_computed` | `session_summary.training_load` where `activity_id` matches |
| `analysis.session.compared` (own) | `session_summary.adherence_pct`, `confidence` |

### Events produced (topic `analysis`)
| Type | Payload | Consumers |
|---|---|---|
| `analysis.session.compared` | `{ session_id, athlete_id, adherence_pct, confidence, path }` | self (projection), gateway (future push) |

### Group dashboard query (MySQL, over projections)
```sql
SELECT s.week_start,
       COUNT(DISTINCT s.athlete_id)          AS athletes,
       AVG(s.training_load)                  AS avg_load,
       AVG(s.adherence_pct)                  AS avg_adherence,   -- NULLs ignored
       SUM(s.status = 'missed')              AS misses
FROM session_summary s
JOIN membership_window m
  ON  m.athlete_id = s.athlete_id
  AND m.group_id   = ?
  AND m.joined_at <= s.scheduled_date
  AND (m.left_at IS NULL OR m.left_at > s.scheduled_date)
WHERE s.assessoria_id = ?
  AND s.scheduled_date BETWEEN ? AND ?
GROUP BY s.week_start
ORDER BY s.week_start;
```

### Rules
- `adherence_pct` NULL never becomes 0 — not in the engine, not in the projection, not in the query.
- Projections are disposable: any bug → rebuild from Kafka, no manual data fixing.
- No `latlng` is ever requested from activity-svc by this service.

### Tests
- Projection replay from empty produces the same `dashboard` numbers as before.
- Athlete moves group in April → March numbers unchanged in Group A.
- Session with `confidence='low'` → excluded from `AVG`, still counted in `total_count`.
- Path A on a 6×800 FIT fixture → 6 repetitions, adherence computed; GPX fixture → `not_compared`.

**Unblocks:** feature blocks 7, 8.

---

## Service dependency order

```
S1 identity-svc ──▶ S2 training-svc ──▶ S4 analysis-svc
        │                  ▲
        └──▶ S3 activity-svc ┘
```

S1 first (everything needs tenants and `membersOn`). S2 and S3 in parallel. S4 last, but its projection consumers can be written as soon as the event contracts in `packages/shared` are agreed — agree those **before** S2/S3 start, not after.

---

## Execution order & sizing

| Order | Block | Depends on | Size |
|---|---|---|---|
| 1 | 0 Foundations (infra is bigger now: 5 apps, Kafka, RabbitMQ) | — | M |
| 2 | 1 Auth & login | 0 | M |
| 3 | 2 Coach people | 1 | M |
| 4 | 3 Groups | 2 | S |
| 5 | 4 Athlete profile + file/manual entry | 1 | M |
| 6 | 5 Activity pipeline | 4 | M–L (no OAuth, but FIT/GPX/TCX parsing) |
| 7 | 6 Prescription/session | 3 | L |
| 8 | 7 Comparison (Path A) | 5, 6 | L |
| 9 | 8 Dashboards (projections) | 7 | M |
| 10 | 9 Watch app (Connect IQ) — **only after pilot** | 5, 6, 7, push notifications | L |

Blocks 2–3 and 4–5 run in parallel once Block 1 is done. Kafka projections (Block 8) can start as soon as the events from Blocks 3, 5, 6 exist.

---

## Open decisions surfaced by this breakdown

1. **Athlete credential** — A or B (Block 1).
2. **Service granularity** — 4 services + gateway is the proposal. If the team is 3–4 people, consider merging training-svc and analysis-svc for MVP; the Kafka boundary between them is the one that costs most and pays least before the pilot.
3. **ORM** — TypeORM vs Prisma, one for all services.
4. **Mobile DB** — SQLite + Drizzle (recommended) vs WatermelonDB.
5. **Who can edit athlete personal data** — coach, athlete, or both.
6. **Timezone** — add `timezone` to `assessoria`; `markMissed` uses it.
7. **D9 — file formats** on day one (proposal: FIT, GPX, TCX; ZIP of those).
8. **Push notifications** — out of scope (UML §8), but "new workout assigned" and "upload reminder" become important precisely because there is no Strava webhook. Flag for post-MVP.

---
---

# Use cases from UML.md — for review

Grouped by package, with actor and the block that implements each. Strava-dependent use cases are listed as **removed** so the numbering still matches `UML.md`.

## Actors
| Actor | Kind | Status |
|---|---|---|
| Athlete | primary, human | runs, uploads/enters activities, views own data |
| Coach | primary, human | prescribes, follows athletes/groups, gives feedback |
| Admin | primary, human | generalization of Coach — inherits every coach UC, plus UC02, UC23 |
| ~~Strava~~ | secondary, system | **removed** |
| Scheduler | secondary, temporal | now only fires UC16 (mark missed) |

## Package 1 — Access & configuration
| UC | Name | Actor | Block | Notes |
|---|---|---|---|---|
| UC01 | Authenticate | Athlete, Coach | 1 | athlete credential missing from model |
| UC02 | Manage coaches | Admin | 2 | |
| UC03 | Register athlete | Coach | 2 | |
| UC04 | Inactivate athlete | Coach | 2 | D7 — never delete |
| UC05 | Manage groups | Coach | 3 | |
| UC06 | Link athlete to group | Coach | 3 | historied membership |
| UC23 | Configure tolerances | Admin | 6 | |
| UC24 | Set athlete parameters | Coach | 2 | **mandatory** on onboarding — no threshold, no load |

## Package 2 — Data integration
| UC | Name | Actor | Block | Status |
|---|---|---|---|---|
| ~~UC07~~ | Connect Strava account | — | — | **removed** |
| ~~UC08~~ | Revoke connection | — | — | **removed** |
| UC09 | Import history → **Import activity file** | Athlete (Coach on behalf) | 4→5 | kept, re-scoped: FIT/GPX/TCX/ZIP from device |
| ~~UC10~~ | Receive activity (webhook) | — | — | **removed** |
| UC11 | Sync activity | System (via UC09 / manual) | 5 | kept: single dedupe point; «include» UC17, UC18 |
| ~~UC12~~ | Refresh token | — | — | **removed** |
| ~~UC25~~ | Reconcile lost activities | — | — | **removed**; replaced by "late upload flips missed → completed" rule (Block 5) |
| **UC26 (new)** | Enter manual activity | Athlete | 4 | `source='manual'`; «include» UC11 |
| **UC27 (new, post-pilot)** | Capture run on watch | Athlete (+ Garmin watch as secondary system actor) | 9 | `source='garmin_ciq'`; «include» UC11; pairing sub-flow |

## Package 3 — Prescription, execution, analysis
| UC | Name | Actor | Block | Relations |
|---|---|---|---|---|
| UC13 | Create prescription | Coach | 6 | |
| UC14 | Assign workout | Coach | 6 | «include» UC16; sessions born here as `planned`; group resolved **on scheduled date** |
| UC15 | Edit prescription | Coach | 6 | |
| UC16 | Mark not done (missed) | Scheduler | 6 | reversible by late upload |
| UC17 | Compare planned vs actual | System (via UC11) | 7 | «extend» UC19 when confidence low; null ≠ zero |
| UC18 | Compute load | System (via UC11) | 5 | pace_v1 now, hr_v1 later |
| UC19 | Review low-confidence comparison | Coach | 7 | extension point only |
| UC20 | View individual dashboard | Athlete, Coach | 8 | |
| UC21 | View group dashboard | Coach | 8 | «include» UC18; date-window projection; no maps |
| UC22 | Record feedback | Coach | 7 | |

## Things in UML.md worth questioning (after removing Strava)
- **UC09 actor** — allow the coach to upload on the athlete's behalf during onboarding (proposed above).
- **Athlete-facing detail screens** (session detail, activity detail) are implied, not named — in Blocks 4, 5, 7.
- **No UC for the athlete editing own profile** — added in Block 4; confirm.
- **UC26 manual activity** is new; confirm it's in MVP. Without Strava it is the only zero-friction path for phone-only athletes.
- **UC16 timing** needs a timezone and grace rule; the "false missed" risk moves from lost webhooks to forgotten uploads — the missed → completed reversal covers it, an upload reminder notification would prevent it.
- **UC02 edge:** last admin cannot be inactivated — not written anywhere; added in Block 2.
