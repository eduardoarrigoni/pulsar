# Pulsar — Mobile Layer: Coach Use Cases

**Source:** `WORK-BLOCKS.md` (blocks 1–9) for pipeline, screens and routes; use case numbers follow `entrega-equipe/UML.md` where they still apply.
**Scope:** MVP, no Strava. React Native app. Athlete flows are out of this file.
**Perspective:** what the coach does on the phone, and what the app does with the gateway in response.

**Model in force (overrides MODEL.md / UML.md / WORK-BLOCKS.md where they differ):**

- Two roles only: **athlete** and **coach**. No admin, no tenant. Every account signs up on its own.
- **Connection** athlete ↔ coach is a friend-style link: either side sends a request by the other's **username** and the other accepts, or the coach shares an **invite link** and whoever opens it becomes their athlete.
- The coach owns **assessorias** and **groups** (containers with membership history). They differ in who can join:
  - **Group: open.** Any athlete can be a member, **connected to the coach or not**. They join by sending a join request the coach accepts, or by opening the group's invite link.
  - **Assessoria: connected only.** Athlete and coach **must be connected** (UC03) first. Only connected athletes can be put in it or join it, and disconnecting removes the athlete from it.
- The coach creates prescriptions and sends them to any number of athletes by **filter** (level, assessoria, group, …) or by **group**.
- Tenant boundary = the connection, plus group membership for that group's sessions only. A coach sees full data only for athletes connected to them. A **group-only member** (in the coach's group, not connected) shows up in that group's roster, receives its workouts, and shares **only the sessions of that group's workouts** with the coach. `coach_id` replaces `assessoria_id` as the scoping key on every query.
- **Rank (`GAMIFICATION.md`)** is two separate 1–100 ladders. The athlete's (UC29–UC32) is earned by running; the coach can see and filter by it, never set it. The coach's own (UC35, UC36) is earned by coaching — roster, prescriptions, feedback, review queue, live sessions — and athletes never see it. Level (UC24) stays the coach's own label, separate from both.
- **Daily missions (UC33)** are assigned to both roles, 3 per local day from separate catalogues, and are the main daily XP source. The Coach Home carries its own Missions card.
- **Live following (UC28)** makes the watch two-way (samples up, coach commands down in the response body), shows the athlete's map and position to the coach during a run the athlete chose to share, and requires push notifications in MVP.

---

## Mobile baseline (coach side)

| Piece | What it holds |
|---|---|
| **Secure storage** | refresh token only |
| **Local DB (SQLite + Drizzle)** — read cache | connected athletes (id, name, username, level, threshold set?), pending requests (received and sent), assessorias, groups + members as of today, prescriptions + blocks for the visible date range, sessions with status, session results, dashboards last fetched, `sync_cursor` per scope |
| **Local DB — outbox** | workout drafts, feedback (`PUT /sessions/:id/feedback`), file uploads on behalf (`POST /athletes/:id/imports`) |
| **API client** | base URL = api-gateway; `Authorization: Bearer <access>`; interceptor renews once on 401 via `POST /auth/refresh`, then retries; second 401 → Login |
| **Sync** | `GET /sync?since=<cursor>` per scope on screen focus and on app foreground |
| **Push** | FCM/APNs: `live_started`, `run_received`, connection requests, group join requests; delivered by the gateway from Kafka events |
| **Live channel** | WebSocket to the gateway (`/live`), opened only while a Live screen (UC28) is on foreground; carries samples in, commands out |
| **Navigation** | Splash → Login / Sign up → Role router → Coach Home with tabs **Athletes · Groups · Workouts · Dashboard · Settings** |
| **Coach Home extras** | rank strip (tier badge, rank, progress bar) and Missions card (3 per day, streak) above the athlete list — see `GAMIFICATION.md` UC33, UC36 |

What the coach app never holds or shows: athlete `latlng`/maps **outside a live session the athlete turned on (UC28)**, athlete credentials, athletes neither connected to this coach nor in one of their groups, and anything about a group-only member beyond that group's sessions (no activities, parameters, individual dashboard or live). Live positions are kept in memory only, never in the local DB.

### Coverage

| UC | Name | Screens | Gateway routes | Works offline |
|---|---|---|---|---|
| UC01 | Sign up / authenticate | Splash, Login, Sign up, Forgot/Reset | `/auth/*`, `/me` | session restore only |
| UC03 | Connect with athlete (request either way, or link) | Requests, Find athlete, Invite link | `/connections/*` | read cache |
| UC24 | Set athlete parameters & level | Athlete parameters | `PUT /athletes/:id/parameters` | no |
| UC04 | Disconnect athlete | Athlete detail | `DELETE /connections/:id` | no |
| UC05 | Manage assessorias & groups | Groups tab | `/assessorias*`, `/groups*` | read cache |
| UC06 | Put athlete in assessoria / group; group join requests & link | Group detail, Add athlete, Join requests, Group invite link | `/groups/:id/members*`, `/assessorias/:id/members*`, `/groups/:id/join-requests*`, `/groups/:id/link`, `POST /groups/join` | no |
| UC23 | Configure tolerances | Settings → Tolerances | `GET/PUT /settings/tolerances` | no |
| UC09 | Import activity file (athlete own, or coach on behalf) | Athlete activities → Import | `POST /athletes/:id/imports`, `GET .../imports/:jobId` (athlete side: `/me/imports`) | queued in outbox |
| UC13 | Create prescription | Workout form | `POST /prescriptions` | draft only |
| UC14 | Send workout (filters / groups) | Workout detail → Send | `POST /prescriptions/:id/send`, `DELETE .../assignments/:aid` | no |
| UC15 | Edit prescription | Workout form | `PATCH /prescriptions/:id` | draft only |
| UC19 | Review low-confidence comparison | Review queue, Session detail | `GET /sessions/review-queue`, `POST /sessions/:id/relink`, `POST /sessions/:id/reviewed` | read cache |
| UC22 | Record feedback | Session detail → Feedback | `PUT /sessions/:id/feedback` | yes (outbox) |
| UC20 | View individual dashboard | Athlete detail → Dashboard | `GET /athletes/:id/dashboard` | read cache |
| UC21 | View group dashboard | Dashboard tab | `GET /groups/:id/dashboard`, `GET /assessorias/:id/dashboard` | read cache |
| UC28 | Follow athlete live & send commands | Athletes tab / Group detail → Live, Live screen | `GET /athletes/:id/live`, WS `/live`, `POST /athletes/:id/live/commands` | no |
| UC32 | See and filter by athlete rank | Athletes tab, Athlete detail, Send workout | `GET /athletes/:id/rank`, `/athletes?rankFrom=&rankTo=` | read cache |
| UC33 | Daily missions (coach catalogue) | Coach Home → Missions card, Mission detail, history | `GET /me/missions`, `POST /me/missions/:id/reroll` | read cache |
| UC35 | Earn experience (coach) | none — awarded from events | — | — |
| UC36 | View own rank & progress | Coach Home → rank strip, Rank detail, XP log | `GET /me/rank`, `GET /me/xp` | read cache |
| UC11, UC16, UC17, UC18, UC26, UC27 | system / athlete | seen through status changes | via sync | read cache |
| ~~UC02~~ | Manage coaches | dropped — no admin role | — | — |
| ~~UC07, UC08, UC10, UC12, UC25~~ | removed with Strava | — | — | — |

---

## Package 1 — Access & connections

### UC01 — Sign up / authenticate

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | none for sign up; coach account exists for login |
| **Postcondition** | tokens stored, Coach Home open, first sync pulled |

**Main flow — sign up**

1. Login screen → "Create account" → role picker: Athlete / **Coach**.
2. Form: name, **username** (unique, lowercase), email, password.
3. `POST /auth/signup { role: 'coach', ... }` → `201` + tokens.
4. Coach Home, empty state: "Find an athlete by username, share your invite link, or wait for requests" → UC03.

**Main flow — login / restore**

1. Splash reads the refresh token from secure storage.
2. Found → `POST /auth/refresh` → new pair → Coach Home.
3. Missing or 401 → Login: email or username + password → `POST /auth/login` → tokens → `GET /me` → Coach Home.
4. Background: `GET /sync?since=0` for athletes, requests, groups, current week's prescriptions and sessions.

**Alternative flows**

- **2a — username taken.** `409` inline; app suggests `name123`-style alternatives.
- **Wrong credentials / inactive account.** Same generic error.
- **Forgot password.** Login → Forgot → email → `POST /auth/password/forgot` → deep link `pulsar://reset/<token>` → Reset → `POST /auth/password/reset` → Login.
- **Access token expires mid-session.** Interceptor refreshes once and replays; refresh fails → wipe tokens → Login.
- **Logout.** Settings → Logout → `POST /auth/logout` → secure storage cleared, local DB wiped, Login.
- **Change username.** Settings → Profile → `PATCH /me { username }`; old username stops resolving immediately.

---

### UC03 — Connect with athlete

| | |
|---|---|
| **Actor** | Coach (athlete initiates in flow A) |
| **Precondition** | both accounts exist |
| **Postcondition** | `connection { coach_id, athlete_id, connected_at, source: 'athlete_request' \| 'coach_request' \| 'link' }` active; athlete visible in Athletes tab |

**Flow A — athlete sends a request, coach accepts**

1. Athlete (their app) → "Find coach" → types the coach's username → `POST /connections/requests { toUsername }`.
2. Coach app, next sync: Athletes tab badge "1 request"; Home card "New request from *name*".
3. Coach → Requests → **Received** → `GET /connections/requests?direction=received` → list: athlete name, username, sent at.
4. Accept → `POST /connections/requests/:id/accept` → connection created; athlete moves to the Athletes list with badge **"threshold not set"**.
5. App opens **Athlete parameters** (UC24) for that athlete.

**Flow B — coach sends a request, athlete accepts**

1. Athletes tab → "Find athlete" → types the athlete's username → `GET /users/lookup?username=` → card with name and avatar.
2. Send request → `POST /connections/requests { toUsername }` → `201`; request appears under Requests → **Sent** with status `pending`.
3. Athlete (their app), next sync: request card → Accept or Decline.
4. Coach app, next sync: accepted → athlete appears in the Athletes list with badge "threshold not set" and a "new" marker; declined → sent request shows `declined`.
5. Coach opens the athlete → **Athlete parameters** (UC24).

**Flow C — coach shares an invite link**

1. Athletes tab → "Invite" → `GET /connections/link` → `pulsar://join/<code>` (+ `https://` fallback) with share sheet / QR.
2. Athlete opens the link → if logged in: `POST /connections/join { code }`; if not: Sign up / Login, then the same call.
3. Connection created with `source: 'link'`, no acceptance step.
4. Coach app, next sync: athlete appears in the list with badge "threshold not set" and a "new" marker until opened.

**Alternative flows**

- **A4a — Coach declines.** `POST /connections/requests/:id/decline` → request closed; athlete can send again.
- **A1a / B1a — username not found.** Generic "user not found"; no enumeration by partial match; lookup only matches the full username.
- **B2a — Cancel a sent request.** Requests → Sent → Cancel → `DELETE /connections/requests/:id` while `pending`.
- **B2b — Pending request already exists in the other direction.** Server accepts it instead of creating a second one → connection created immediately.
- **B2c — Athlete has blocked requests.** `201` returned, request never delivered (no signal to the coach).
- **C1a — Regenerate link.** "Reset link" → `POST /connections/link/rotate` → old code stops working; existing connections untouched.
- **Already connected.** Request or join returns `200` no-op; nothing duplicated.
- **Athlete connected to another coach too.** Allowed; each coach sees only their own prescriptions, sessions and feedback for that athlete. Activities are the athlete's and are visible to every connected coach.
- **Offline.** Requests list from cache; Accept/Decline and link generation disabled.

---

### UC24 — Set athlete parameters & level

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | athlete connected |
| **Postcondition** | `threshold_pace_spk`, `max_heart_rate`, `resting_heart_rate`, `level` stored per (coach, athlete); badge cleared |

**Main flow**

1. Entry from UC03 step A5/B5/C4, or Athlete detail → Training parameters → Edit.
2. Threshold pace typed as `m:ss /km` (stored as seconds), max HR, resting HR, **level** (`beginner` / `intermediate` / `advanced`, coach-defined labels allowed later).
3. Client validation: pace 120–1200 s/km, HR 30–230, resting < max.
4. Save → `PUT /athletes/:id/parameters` → `200`.
5. Athlete detail shows the values; list row loses the "threshold not set" badge; `level` becomes a filter chip in the Athletes tab and in UC14.

**Alternative flows**

- **4a — out of range.** `422` inline.
- **Changing an existing threshold.** Future activities use the new value; past loads unchanged, activity detail shows the threshold used.
- **Offline.** Save disabled.

---

### UC04 — Disconnect athlete

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | connection active |
| **Postcondition** | `connection.ended_at` set; open **assessoria** memberships closed; **group** memberships kept (athlete becomes a group-only member); future `planned` sessions from this coach removed except those of groups the athlete is still in; history kept |

**Main flow**

1. Athlete detail → ⋯ → Disconnect.
2. Confirm: "History is kept. The athlete will no longer receive your workouts."
3. `DELETE /connections/:id` → `200`.
4. Athlete leaves the Athletes list (visible under "Past athletes" filter, read-only).
5. Assessoria details (as of today) no longer list the athlete; membership history shows `left_at = today`.
6. Group details still list the athlete, now marked **"not connected"**: the coach keeps seeing that group's sessions only (see model).

**Alternative flows**

- **Athlete disconnects first (their app).** Coach app, next sync: same result as steps 4–6 plus a Home card "*name* left".
- **Coach wants the athlete out of the groups too.** Remove them from each group (UC06 step 5). Disconnect does not do it.
- **Reconnect.** Either side runs UC03 again → new connection row; assessoria memberships **not** reopened, past sessions and results still linked to the old connection.
- **Pending imports for the athlete.** Cancelled; progress card shows "cancelled".

---

### UC05 — Manage assessorias & groups

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | coach logged in |
| **Postcondition** | assessoria or group created, renamed or archived |

Assessoria = top-level container (e.g. the coach's team or brand); group = smaller set, optionally inside an assessoria. Both have dated membership.

| | Group | Assessoria |
|---|---|---|
| Who can be a member | any athlete account | only athletes **connected** to this coach |
| How they get in | join request accepted by the coach, group invite link, or added by the coach (connected athletes) | added by the coach, or join request / link **from a connected athlete** |
| Coach sees | connected member: everything; group-only member: that group's sessions only | everything (all members are connected) |
| On disconnect (UC04) | membership kept | membership closed |

A group inside an assessoria is still open: joining the group does **not** put the athlete in the assessoria, and group-only members of that group do not count in the assessoria's dashboard.

**Main flow**

1. Groups tab → two sections: Assessorias, Groups → `GET /assessorias`, `GET /groups` → each with member count as of today.
2. "+" → type (assessoria / group), name, parent assessoria (group only, optional) → `POST /assessorias` or `POST /groups` → detail.
3. Detail → Rename → `PATCH /:type/:id`.
4. Detail → Archive → confirm → `PATCH /:type/:id { active: false }` → open memberships closed with `left_at = today`; item leaves the active list.

**Alternative flows**

- **Archive assessoria with groups inside.** Groups stay; their parent shows "archived".
- **Offline.** Lists and details from cache; writes disabled.

---

### UC06 — Put athlete in assessoria / group

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | container active. **Assessoria:** athlete connected. **Group:** any athlete account |
| **Postcondition** | membership row with `joined_at`, `source: 'coach_added' \| 'join_request' \| 'link'`; or `left_at` on removal |

**Flow A — coach adds athletes**

1. Group (or assessoria) detail → members as of a date (default today) → `GET /groups/:id/members?on=YYYY-MM-DD`. Group-only members carry a **"not connected"** chip.
2. Add → picker lists connected athletes **not** currently in the container, with level chips and search by name/username; multi-select.
3. `joined_at` defaults to today, backdating allowed, one date for the whole selection.
4. Confirm → `POST /groups/:id/members { athleteIds[], joinedAt }` → list refreshes.
5. Remove → tap member → Remove → `left_at` defaults today → `POST /groups/:id/members/:mid/leave { leftAt }` → member leaves the "as of today" list; history keeps the row.

**Flow B — athlete asks to join a group (no connection needed)**

1. Athlete (their app) → "Find group" → coach's username → the coach's **groups** only (assessorias are never listed to a non-connected athlete) → Ask to join → `POST /groups/:id/join-requests`.
2. Coach app, next sync: Group detail badge "1 join request"; push.
3. Group detail → Join requests → `GET /groups/:id/join-requests` → Accept → `POST /groups/:id/join-requests/:rid/accept` → membership `source: 'join_request'`, `joined_at = today`. **No connection is created.**
4. Decline → `POST /groups/:id/join-requests/:rid/decline`; athlete can ask again.

**Flow C — group invite link**

1. Group detail → "Invite to group" → `GET /groups/:id/link` → `pulsar://group/<code>` (+ `https://` fallback), share sheet / QR.
2. Athlete opens it → logged in: `POST /groups/join { code }`; if not: Sign up / Login, then the same call.
3. Membership created with `source: 'link'`, no acceptance step, **no connection created**.
4. "Reset link" → `POST /groups/:id/link/rotate` → old code stops working; members stay.

This link is not the coach's connection link (UC03 flow C). The connection link connects; the group link only adds to that group.

**Flow D — join an assessoria**

1. Only a connected athlete can ask or be added. Athlete (their app) → My coaches → coach → Assessorias → Ask to join → same accept/decline as flow B on `/assessorias/:id/join-requests`.
2. Not connected → the app shows "Connect with this coach first" and starts UC03 flow A.

**Alternative flows**

- **3a — future `joined_at`.** Picker caps at today; server `422`.
- **4a — athlete already has an open membership here.** Skipped in the response (`added / skipped` counts).
- **B/C — athlete already a member.** `200` no-op.
- **Add a group-only member to an assessoria.** Server `409 "not connected"`. The coach must connect first (UC03), e.g. with "Invite to connect" from the member row.
- **Group-only member later connects.** Membership unchanged; the "not connected" chip goes away and the coach sees full data from then on.
- **Athlete leaves a group (their app).** `POST /groups/:id/members/me/leave` → `left_at = today`; that group's future `planned` sessions are removed.
- **Coach removes a group-only member.** Same as step 5; the athlete no longer sees the group.
- **From Athlete detail.** Athlete detail → Groups → "Add to…" → same call with one athlete.
- **Historical view.** Change the date in the detail → same route with another `on=`, read-only.
- **Timeline.** Athlete detail → Groups → `GET /athletes/:id/memberships` → joined/left per container.

---

### UC23 — Configure tolerances

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | coach logged in |
| **Postcondition** | one `tolerance_setting` per `workout_type` for this coach |

**Main flow**

1. Settings → Tolerances → `GET /settings/tolerances` → table per workout type with defaults seeded from PRESCRIPTION.md §9.
2. Edit pace faster %, pace slower %, distance % per row.
3. Save → `PUT /settings/tolerances` → `200`.
4. Comparisons of this coach's sessions from now on use the new values; past results keep their `tolerance_used`.

**Alternative flows**

- **Timezone.** Settings → Profile → timezone → `PATCH /me { timezone }`; drives UC16 for this coach's sessions from the next hourly run.
- **Offline.** Save disabled.

---

## Package 2 — Data integration (coach side)

### UC09 — Import activity file

| | |
|---|---|
| **Actor** | Athlete (own files, `POST /me/imports`) **or** Coach (on the athlete's behalf, `POST /athletes/:id/imports`) |
| **Precondition** | athlete connected (coach flow) |
| **Postcondition** | `import_job` created; activities appear in the athlete's list as the job completes; same pipeline whoever uploaded |

**Main flow — coach uploads for the athlete**

1. Athlete detail → Activities → Import.
2. File picker: `.fit`, `.gpx`, `.tcx`, or a `.zip` of those.
3. Upload → `POST /athletes/:id/imports` (multipart) → `202 { jobId }`.
4. Progress card: `GET /athletes/:id/imports/:jobId` polled → `total / imported / duplicates / failed`.
5. `done` → card collapses; activity list refreshes via sync.
6. Each imported activity links or creates a session (UC11); statuses update in Workout detail and dashboards on the next sync.

**Main flow — athlete uploads their own file (as the coach sees it)**

1. Athlete (their app) picks a file or shares it from the watch app → `POST /me/imports` → same job pipeline.
2. Coach app, next sync: new activities in Athlete activities; `import_job.requested_by = athlete:<id>` shown as "uploaded by athlete" in activity detail.
3. Sessions link, compare and load exactly as in the coach flow.

**Alternative flows**

- **Offline.** File copied to app storage, outbox row, card "queued"; upload on reconnect.
- **Both upload the same file.** Second one counts as duplicate; one activity.
- **4a — some files failed.** Job still `done`; tap → per-file errors.
- **4b — duplicates.** Counted, not errors.
- **4c — job `failed`.** Retry re-submits the same file.
- **Athlete disconnected while queued.** Job cancelled; card "cancelled".

---

### Athlete activity list / detail (coach view)

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | athlete connected |
| **Postcondition** | none (read-only) |

**Main flow**

1. Athlete detail → Activities → `GET /athletes/:id/activities?from&to` → date, type, distance, duration, source (`file_upload` / `manual` / `garmin_ciq`), load or "threshold not set".
2. Tap → `GET /athletes/:id/activities/:aid` → summary, laps, pace/HR charts, linked session (if any, only this coach's). **No map.**
3. Linked session → Session detail.

**Alternative flows**

- **Activity linked to another coach's session.** Shown as "linked to another coach's plan"; no result visible.
- **Offline.** From cache.

---

## Package 3 — Prescription, execution, analysis

### UC13 — Create prescription

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | coach logged in |
| **Postcondition** | `workout_prescription` + `workout_block` rows; if recipients were chosen, one `planned` session per athlete |

**Main flow**

1. Workouts tab → "+".
2. Form (PRESCRIPTION.md §3): header — name, workout type, scheduled date, notes, **send to** (optional, see UC14 picker); warm-up (optional); main set (required); cool-down (optional). Paces `m:ss`, stored as seconds.
3. Every change autosaves a draft in the local DB.
4. Client validation (PRESCRIPTION.md §4) on Save; errors inline.
5. Save → `POST /prescriptions` → `201 { id }`.
6. Recipients chosen → `POST /prescriptions/:id/send` (UC14) → `{ sessionsCreated, skipped }`.
7. Draft deleted → Workout detail.

**Alternative flows**

- **Free-text toggle.** `is_structured = false`; sessions still created; never compared.
- **Duplicate.** Workout detail → Duplicate → form prefilled, new date, no recipients.
- **Offline.** Draft kept; Save disabled; "Continue draft" prompt on next open with connection.
- **5a — server validation fails.** `422` mapped into the form.
- **6a — send fails after create succeeded.** Detail opens "not sent yet"; retry from the detail.
- **App killed mid-edit.** Draft restored.

---

### UC14 — Send workout (filters / groups)

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | prescription exists |
| **Postcondition** | `prescription_assignment` rows (one per selection rule) and one `session` `planned` per resolved athlete |

**Main flow**

1. Workout detail → Send.
2. Recipient picker, three ways combinable in one send:
   - **Groups / assessorias** — one or many; resolved to members **on `scheduled_date`**.
   - **Filters** — level (multi), assessoria, group, "threshold set", "no workout that day"; preview shows the resolved athlete list and count.
   - **Athletes** — manual multi-select with search by name/username.
3. Preview: "Will create N sessions" with the resolved names; coach can untick individuals.
4. Confirm → `POST /prescriptions/:id/send { groupIds[], assessoriaIds[], filters, athleteIds[], excludeIds[] }` → `200 { sessionsCreated, skipped }`.
5. Workout detail lists every athlete with status `planned`, grouped by how they were selected.
6. Athletes' apps receive the session on their next sync.

**Alternative flows**

- **Group resolved on the scheduled date.** An athlete who joins after `scheduled_date` gets no session; one who left before gets none.
- **Group-only members.** A **group** selection includes them. **Filters** and **Athletes** pick only connected athletes: level and threshold are set per connection (UC24), and a coach cannot send an individual workout to someone they are not connected to. Their sessions have no load until they connect (no threshold).
- **Filter is a snapshot.** Athletes matching the filter later do not get the workout; coach sends again.
- **Athlete selected twice (group + filter).** One session; `skipped` counts it.
- **Same recipients sent twice.** No duplicates; `sessionsCreated: 0`.
- **Send later to more athletes.** Repeat steps 1–4; new assignments appended.
- **Remove an assignment.** Workout detail → assignment → Remove → `DELETE /prescriptions/:id/assignments/:aid`. Enabled only while all its sessions are `planned`; otherwise disabled with "sessions already completed or missed".
- **Remove one athlete.** Workout detail → athlete → Remove → `DELETE /sessions/:sid` (only while `planned`).
- **Offline.** Send disabled.

---

### UC15 — Edit prescription

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | prescription exists |
| **Postcondition** | prescription/blocks updated; `planned` sessions moved if the date changed; completed results flagged `stale` |

**Main flow**

1. Workout detail → Edit → same form, prefilled.
2. Changes autosave to a draft.
3. Save → `PATCH /prescriptions/:id` → `200`.
4. Date changed → every `planned` session moves; group-based assignments are **not** re-resolved for the new date.
5. Blocks changed → sessions already `completed` keep their result and enter the Review queue as **stale**.
6. Draft deleted → Workout detail.

**Alternative flows**

- **Offline.** Draft kept, Save disabled.
- **Discard.** Draft deleted.

---

### UC19 — Review low-confidence comparison

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | a `session_result` of this coach with `confidence = 'low'`, `ambiguity_flag = true` or `stale = true`, not reviewed |
| **Postcondition** | session relinked and re-compared, or marked reviewed |

**Main flow**

1. Dashboard tab badge with queue size → Review queue → `GET /sessions/review-queue`.
2. List: athlete, date, reason (`low confidence` / `ambiguous` / `stale`).
3. Tap → Session detail (`GET /sessions/:id`): prescribed vs achieved per repetition, `confidence` badge, **"could not compare"** where `adherence_pct` is null.
4. **Relink** → picker of this coach's other prescriptions for that athlete on that date → `POST /sessions/:id/relink { prescriptionId }` → `202` → detail updates on next sync.
5. **Mark reviewed** → `POST /sessions/:id/reviewed` → item leaves the queue.

**Alternative flows**

- **4a — no other prescription that day.** Relink hidden.
- **Still `low` after relink.** Stays in the queue with the new result.
- **Offline.** Read-only.

---

### UC22 — Record feedback

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | session belongs to this coach |
| **Postcondition** | `session.coach_feedback` stored; visible to the athlete |

**Main flow**

1. Session detail → Feedback → text (prefilled if any).
2. Save → outbox row → local row shows text with "sending".
3. Outbox → `PUT /sessions/:id/feedback` → `200` → marker cleared.
4. Athlete sees it on next sync.

**Alternative flows**

- **Offline.** Steps 1–2; step 3 on reconnect; same session id overwrites, never duplicates.
- **Edit.** Last write wins.
- **Session gone (athlete disconnected and session was `planned`).** `404` → outbox row dropped.

---

### UC20 — View individual dashboard

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | athlete connected |
| **Postcondition** | none |

**Main flow**

1. Athlete detail → Dashboard → period (default last 4 weeks).
2. `GET /athletes/:id/dashboard?from&to` → per week: planned / completed / missed (this coach's sessions), weekly load (all activities), adherence "n of m compared".
3. Tap week → sessions with status and adherence.
4. Tap session → Session detail.
5. Cached per athlete + period.

**Alternative flows**

- **No threshold.** Load shows "threshold not set"; excluded from weekly sums.
- **No compared sessions.** Adherence "—", never 0 %.
- **Offline.** Last cached period with timestamp.

---

### UC21 — View group dashboard

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | group or assessoria exists |
| **Postcondition** | none |

**Main flow**

1. Dashboard tab → container picker (group or assessoria) + period (default last 8 weeks) + optional level filter.
2. `GET /groups/:id/dashboard?from&to&level=` (or `/assessorias/:id/dashboard`) → per week: athletes counted, average load, average adherence, misses.
3. Tap week → athletes who were members that week (membership window applied server-side) with their own numbers.
4. Tap athlete → UC20 (connected); group-only member → only that group's sessions, no individual dashboard.
5. Cached per container + period + filter.

**Alternative flows**

- **Container younger than the period.** "Needs a few weeks of data".
- **Athlete moved containers mid-period.** Counted in each only for weeks inside their membership window.
- **Group-only members in a group dashboard.** Counted for completed / missed and adherence of that group's sessions; left out of average load (no threshold).
- **Offline.** Last cached view.

---

## Package 4 — Live

### UC28 — Follow athlete live & send commands

| | |
|---|---|
| **Actor** | Coach |
| **Secondary** | Garmin watch (Pulsar data field, Block 9), athlete's phone |
| **Precondition** | athlete connected; athlete has a paired watch with live sharing **on**; a run is in progress |
| **Postcondition** | none persisted beyond the live session log (`live_session`, `live_command`); the run itself arrives later as a normal activity (UC27) |

**What the watch sends during a live run**

- Every 5–10 s: `POST /ciq/live/:runId/samples { seq, t, lat, lng, dist_m, pace_spk, hr, lap, elapsed_s }`.
- Response is `200 { commands: [...] }` — pending coach commands since the last sample. The field shows each one as an alert (vibration + text, ~10 s) and marks it delivered on the next post.
- First sample opens the live session; no sample for 2 min closes it; `finish` (UC27) closes it.

**Main flow**

1. Coach opens the app. Athletes tab rows and Group detail members show a pulsing **LIVE** badge for anyone running with live sharing on (`GET /athletes/live` on tab focus; push `live_started` if the app is closed).
2. Coach finds the athlete — Athletes list (search / level filter) or Groups → group → member — and taps **Follow live**.
3. App calls `GET /athletes/:id/live` → `{ runId, startedAt, plannedRoute?, session?, lastSample }` and opens the WebSocket `/live?runId=` → samples stream in.
4. **Live screen**:
   - Map: current position (dot with heading), track so far (solid line), **intended route** (dashed line, from the route the athlete attached before starting, or the prescription's route if it has one), auto-follow toggle.
   - Header: athlete name, elapsed time, distance, current lap.
   - Tiles: **current pace** (30 s rolling), lap pace, **heart rate** with zone colour (from `max_heart_rate` / `resting_heart_rate` set in UC24), cadence and altitude if the watch sends them.
   - Prescription strip: the session planned for today (if any) with the current block highlighted and its target pace; current pace coloured against the tolerance (UC23).
   - Sample age indicator ("2 s ago" / "no signal 40 s").
5. **Send command** bar: `Slow down` · `Speed up` · `Hold pace` · `Next block` · `Stop` · `Message…` (short free text, ≤ 60 chars). Optional value on pace commands (e.g. "to 4:45").
6. Tap → `POST /athletes/:id/live/commands { runId, type, value?, text? }` → `201 { id }` → command shown in a timeline under the map as `sent`.
7. Delivery:
   - **Watch**: included in the response of the athlete's next sample post → alert on the wrist → next post carries `ack: [commandId]` → timeline shows `seen on watch`.
   - **Phone**: gateway sends a push to the athlete's device at the same time → notification "Coach: slow down to 4:45" → timeline shows `delivered to phone`.
8. Coach leaves the screen → WebSocket closed; badge stays until the session ends.
9. Session ends → screen shows "Run finished"; a few minutes later the activity and session result arrive via UC27/UC11 and the Live screen offers "Open session".

**Alternative flows**

- **1a — no one is live.** No badges; Athletes tab has no live section.
- **2a — athlete has live sharing off or no watch.** Follow live disabled with "not sharing live".
- **3a — coach opens the screen mid-run.** Track so far is loaded from `lastSamples` in the `GET`, then streams.
- **4a — no intended route.** Map shows position and track only; strip shows "no route".
- **4b — no prescription today.** Prescription strip hidden; pace has no colour.
- **4c — no HR.** HR tile shows "—".
- **6a — watch offline (phone out of BLE range / no data).** Command stays `sent`; delivered on the next successful post; after 5 min without a post the timeline marks it `expired` and only the phone notification remains.
- **6b — athlete's phone has no push token.** Timeline shows `watch only`.
- **Several coaches follow the same athlete.** Each sees the same stream; each command shows its author name on the watch and phone.
- **Athlete turns live sharing off mid-run (from the watch field or phone).** Stream stops; screen shows "sharing stopped"; commands disabled.
- **Coach loses connection.** WebSocket reconnects with `since=<seq>`; missed samples backfilled.
- **Rate limit.** Max one command per 10 s per coach per run; the bar disables between sends.

**Athlete side (as it affects the coach)**

- Live sharing is a toggle in the athlete's app (default off) and a setting on the data field; the watch sends samples only when both are on.
- Before a run, the athlete can attach an intended route (draw on the map, pick a previous activity's track, or accept the prescription's route). This is the dashed line in step 4.
- Commands appear on the watch as an alert and on the phone as a notification; the athlete has no reply action in MVP.

**Data kept**

- `live_session (id, run_id, athlete_id, started_at, ended_at, sample_count)` and `live_command (id, live_session_id, coach_id, type, value, text, sent_at, seen_on_watch_at, delivered_phone_at)` in activity-svc.
- Samples are kept in memory / short-lived storage for the duration of the run plus 1 h for reconnect backfill, then discarded; the FIT/CIQ upload is the record of the run.

---

## System and athlete use cases — what the coach sees

No coach action. Each lists the state change that reaches the coach app through sync.

### UC11 — Sync activity

1. Athlete uploads a file (UC09), enters an activity manually (UC26) or finishes a run on the watch (UC27).
2. Coach app, next sync: new row in Athlete activities; this coach's session for that athlete + date flips `planned`/`missed` → `completed`; no matching prescription from this coach → session `unplanned` under Athlete detail → Recent sessions.
3. Two prescriptions from this coach on that date → linked to the closest by distance; Review queue as `ambiguous`.
4. Athlete has prescriptions from two coaches on that date → each coach's session is linked independently.

### UC16 — Mark not done (missed)

1. Hourly job in the coach's timezone marks `planned` sessions with `scheduled_date < today` as `missed`.
2. Coach app, next sync: `missed` in Workout detail, Athlete detail, UC20 and the UC21 misses column.
3. Late upload → same session back to `completed`; the miss disappears.

### UC17 — Compare planned vs actual

1. Runs after every link and relink.
2. Coach app, next sync: Session detail shows repetitions prescribed vs achieved, `adherence_pct`, `confidence`.
3. `low` → Review queue. `not_compared` (free-text, manual activity, no usable laps) → "could not compare", no adherence.

### UC18 — Compute load

1. Runs after every `sync()`, using the parameters set by the coach who receives the session; no connected coach with a threshold → load null.
2. Coach app, next sync: `training_load` on the activity and in weekly sums.
3. "Threshold not set" until UC24; past activities stay without load.

### UC26 — Enter manual activity (athlete)

1. Athlete records date/time, type, distance, duration, optional avg HR.
2. Coach app, next sync: activity with source `manual`, no laps, no charts; linked session `completed` with `not_compared`; load computed if threshold set.

### UC27 — Capture run on watch (post-pilot, Block 9)

1. Watch data field posts the run; activity created with source `garmin_ciq`.
2. Coach app: same as UC11 — activity row, session `completed`, comparison, Review queue if `low`; Athletes list row shows "new run" until opened.
