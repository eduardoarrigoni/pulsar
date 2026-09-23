# Pulsar — Rank & Experience

**Scope:** progression for both roles. Athlete rank 1 → 100 and coach rank 1 → 100, each with its own XP sources and its own curve. Rank tiers are numbered (Tier 1–6); the named ladders are Mastery (UC38).
**Numbering continues** `MOBILE.md` (UC01–UC28). Here: UC29–UC38 (UC37 achievements, UC38 Mastery). Athlete groups continue in `SOCIAL.md` (UC39–UC43).
**Owner:** `gamification-svc` (new). Consumes Kafka only; owns `gamification_db`. No other service reads or writes rank.

| | Athlete ladder | Coach ladder |
|---|---|---|
| Use cases | UC29 earn · UC31 view | UC35 earn · UC36 view |
| Shared | UC30 rank up · UC33 daily missions | |
| Earned from | running: distance, pace, constancy, missions, places | coaching: roster, prescriptions, feedback, review, live, missions |
| Rank tiers | Tier 1 → Tier 6 | Tier 1 → Tier 6 |
| Mastery (yearly) | Starter → Legend | Whistle → Maestro |

---

## Model — athlete

- **Rank** is an integer 1–100. Every new athlete account starts at **rank 1** with **0 XP**.
- **XP** only ever increases. Rank never drops, never decays, is never reset.
- XP belongs to the **athlete**, not to a coach–athlete connection. An athlete with two coaches, or none, keeps one rank.
- Rank is cosmetic: it never changes what the app allows, never feeds adherence, load or dashboards.

### Rank curve

`R` = the rank the athlete holds now. `xp_to_next(R)` = the XP for that one step, from `R` to `R + 1`. It is not a running total.

```
xp_to_next(R) = 100 + (R - 1) × 40          R = 1 … 99
```

Step 1 → 2 costs 100 XP; every following step costs 40 XP more than the one before it.

Total XP to hold rank `R`:

```
total_xp(R) = 100 × (R - 1) + 20 × (R - 1) × (R - 2)
```

Rank from an XP total (what UC30 evaluates):

```
rank(xp) = 1 + floor( ( sqrt(6400 + 80 × xp) - 80 ) / 40 ),  capped at 100
```

| Step | XP for that step | Total XP to hold the higher rank |
|---|---|---|
| 1 → 2 | 100 | 100 |
| 10 → 11 | 460 | 2,800 |
| 25 → 26 | 1,060 | 14,500 |
| 50 → 51 | 2,060 | 54,000 |
| 75 → 76 | 3,060 | 118,500 |
| 99 → 100 | 4,020 | 203,940 |

Total rank 1 → 100: **203,940 XP**.

### Tiers

| Ranks | Tier | Badge |
|---|---|---|
| 1–10 | Tier 1 | outline |
| 11–30 | Tier 2 | solid |
| 31–50 | Tier 3 | solid + bar |
| 51–70 | Tier 4 | filled |
| 71–90 | Tier 5 | filled + bar |
| 91–100 | Tier 6 | filled + mark |

Tiers are numbered, not named. The names belong to Mastery (UC38).

---

## Model — coach

- **Rank** is an integer 1–100. Every new coach account starts at **rank 1** with **0 XP**.
- Same permanence rules: XP only rises, rank never drops, never decays, never resets.
- Coach XP is earned from **coaching work and from what the coach's athletes do with it** — never from the coach's own running. A coach who also runs keeps two separate accounts with two separate ranks.
- Rank is cosmetic: it never gates a feature, never appears in an athlete's dashboard numbers, never ranks coaches against each other in a public list.
- Roster-driven sources are capped **per day, not per athlete**, so a coach with 60 athletes does not outrank a coach with 8 by roster size alone.

### Rank curve

```
xp_to_next(R) = 120 + (R - 1) × 50          R = 1 … 99

total_xp(R)   = 25 × (R - 1)² + 95 × (R - 1)

rank(xp)      = 1 + floor( ( sqrt(9025 + 100 × xp) - 95 ) / 50 ),  capped at 100
```

| Step | XP for that step | Total XP to hold the higher rank |
|---|---|---|
| 1 → 2 | 120 | 120 |
| 10 → 11 | 570 | 3,375 |
| 25 → 26 | 1,320 | 17,880 |
| 50 → 51 | 2,570 | 64,675 |
| 75 → 76 | 3,820 | 141,080 |
| 99 → 100 | 5,020 | 254,430 |

Total rank 1 → 100: **254,430 XP**.

### Tiers

| Ranks | Tier | Badge |
|---|---|---|
| 1–10 | Tier 1 | outline |
| 11–30 | Tier 2 | solid |
| 31–50 | Tier 3 | solid + bar |
| 51–70 | Tier 4 | filled |
| 71–90 | Tier 5 | filled + bar |
| 91–100 | Tier 6 | filled + mark |

Same numbered tiers as the athlete ladder; the coach names belong to Mastery (UC38).

---

## UC29 — Earn experience (athlete)

| | |
|---|---|
| **Actor** | System (`gamification-svc`) |
| **Trigger** | `activity.imported`, `activity.load_computed`, daily/weekly/monthly/yearly rollups, UC33, UC34 |
| **Precondition** | athlete account active |
| **Postcondition** | one `xp_entry` row per award; `athlete_progress.xp` increased; UC30 evaluated |

### Sources

| # | Source | Award | Cap |
|---|---|---|---|
| 1 | **Distance** | 10 XP per km (moving distance) | 400 XP per activity |
| 2 | **Activity logged** | 20 XP flat | first 2 activities per calendar day |
| 3 | **Personal best** per distance bucket (1 km, 5 km, 10 km, 21.1 km, 42.2 km) | 150 XP | once per bucket per 30 days |
| 4 | **Threshold pace improved** | 25 XP per second/km removed | 500 XP per change |
| 5 | **Week constancy** — ISO week (Mon–Sun) with ≥ 3 activities | 100 XP, +25 per consecutive qualifying week, capped at 300 | once per week |
| 6 | **Month constancy** — calendar month with ≥ 12 activities | 400 XP | once per month |
| 7 | **Year constancy** — calendar year with ≥ 40 active weeks (week with ≥ 1 activity) | 3,000 XP | once per year |
| 8 | **Activity count milestones** — 10th, 25th, 50th, 100th, 250th, 500th, 1000th activity | 100 · 200 · 300 · 600 · 1,200 · 2,000 · 4,000 XP | once each, lifetime |
| 9 | **Daily mission** | 30 / 60 / 120 XP by difficulty, +50 daily clear | see UC33 |
| 10 | **Unlock a new place** | see UC34 | see UC34 |

**Main flow — per activity (sources 1–4, 8)**

1. `activity.imported` arrives with `athlete_id`, `distance_meters`, `moving_seconds`, `source`, `started_at`.
2. Idempotency: `xp_entry` has `UNIQUE (subject_type, subject_id, source_kind, source_ref)`; a replayed event awards nothing twice.
3. **Distance** — `floor(distance_km × 10)` XP, capped at 400.
4. **Activity logged** — 20 XP if fewer than 2 activities already counted for that local day.
5. **Personal best** — best rolling effort per bucket is computed from `activity_stream`; a bucket whose time beats the athlete's stored best, and whose last award for that bucket is older than 30 days, awards 150 XP and updates `personal_best`.
6. **Activity count** — the athlete's lifetime activity count crossing a milestone awards that milestone once.
7. Entries are written in one transaction; `athlete_progress.xp` is incremented; UC30 runs.
8. Mission progress (UC33) is evaluated from the same event.

**Main flow — constancy (sources 5–7)**

1. Rollup job runs hourly in the athlete's timezone.
2. A finished ISO week with ≥ 3 activities awards 100 XP plus 25 per consecutive qualifying week before it, capped at 300; `week_streak` increments. A week below 3 sets `week_streak = 0` and awards nothing.
3. A finished calendar month with ≥ 12 activities awards 400 XP.
4. A finished calendar year with ≥ 40 active weeks awards 3,000 XP.
5. Each award is one `xp_entry` keyed by period, so a rerun awards nothing twice.

**Main flow — threshold pace (source 4)**

1. `identity.athlete.parameters_changed` arrives with the new `threshold_pace_spk`.
2. New value lower than the stored previous value → `(previous − new) × 25` XP, capped at 500.
3. Higher or equal → no award, no penalty.

**Alternative flows**

- **Manual activity** (`source = 'manual'`). Distance XP at **50 %**; no personal best; counts for sources 2, 5, 6, 7, 8.
- **Duplicate file.** No activity is created, so no XP.
- **Activity deleted or superseded** (CIQ run replaced by its FIT upload). The superseded activity's entries are reversed with a negative `xp_entry`; the replacement awards normally. `athlete_progress.xp` can fall in this one case; **rank still never falls**.
- **Activity with no stream** (manual, or GPX without usable data). Sources 1, 2, 5–8 only.
- **Athlete without a coach.** Everything in this file still applies; rank does not need a connection.
- **Backdated import** (an athlete uploading years of history). Distance, activity-logged and milestone XP are awarded; constancy weeks/months/years older than 90 days are **not** back-awarded; personal bests are recorded without XP.
- **Athlete changes timezone.** Day and week boundaries follow the new timezone from the next rollup; no period is awarded twice.

---

## UC30 — Rank up

| | |
|---|---|
| **Actor** | System |
| **Subject** | athlete (UC29) or coach (UC35) |
| **Precondition** | that subject's `xp` increased |
| **Postcondition** | `rank` raised to the highest rank the XP total covers; `rank.changed` event; notification queued |

**Main flow**

1. After every XP award, the service resolves the rank from the subject's XP total using that role's `rank(xp)` formula.
2. New rank equals the stored rank → nothing happens.
3. New rank is higher → `rank` is updated, one `rank_history` row per rank passed.
4. Kafka `gamification.rank.changed { subject_type, subject_id, from, to, tier_changed }`.
5. Push: "Rank 24 reached". Tier crossed → "Tier 3 reached".
6. The next app open shows the rank-up screen once, then marks it seen.

**Alternative flows**

- **Several ranks at once** (a long backdated import; a coach's first week with a full roster). One rank-up screen showing `from → to`, one push, one row per rank in `rank_history`.
- **Rank 100 reached.** XP keeps accumulating and stays visible; no rank above 100.
- **Negative correction** (superseded activity). XP falls, rank stays; the next awards fill the gap before the next rank-up.

---

## UC31 — View rank & progress (athlete)

| | |
|---|---|
| **Actor** | Athlete |
| **Precondition** | logged in |
| **Postcondition** | none (read-only) |

**Screens**

| Screen | Content |
|---|---|
| Athlete Home — rank strip | tier badge, rank number, progress bar to the next rank, XP remaining |
| Rank detail | big badge, `xp` total, `xp_to_next`, tier ladder with the next tier marked, `rank_history` timeline |
| XP log | reverse-chronological `xp_entry` list: source, amount, date, linked activity or period |
| Rank-up screen | shown once after UC30: `from → to`, tier if crossed, the awards that caused it |
| Streaks card | current week streak, active weeks this year, activities this month against the month's target |

**Main flow**

1. Athlete Home → rank strip → `GET /me/rank` → `{ rank, tier, xp, xpToNext, weekStreak, activeWeeks, monthCount }`.
2. Tap → Rank detail.
3. "How XP works" → the source table of UC29 with the athlete's own totals per source.
4. XP log → `GET /me/xp?before=` → paged entries; tap an activity entry → activity detail.

**Alternative flows**

- **Offline.** Rank, XP and the last page of the log come from the local cache; the strip shows its timestamp.
- **New account.** Rank 1, 0 XP, "Log your first run to start" instead of the bar.
- **Award arrives while the screen is open.** Strip animates the bar; rank-up screen queued until the athlete leaves the current screen.

---

## UC32 — Coach sees and filters by athlete rank

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | athlete connected |
| **Postcondition** | none (read-only) |

**Main flow**

1. Athletes tab — each row shows the tier badge and rank number next to the level chip.
2. Athlete detail — header shows rank, tier and week streak; tap → the athlete's rank detail, read-only, no XP log.
3. Send workout (UC14) — filters gain **rank range** (`from`–`to`) beside level.
4. Assessoria detail and assessoria dashboard — members sortable by rank.

**Alternative flows**

- **Coach cannot change an athlete's rank.** No route exists.
- **Athlete hides rank.** Athlete setting "show my rank to my coaches" (default on); off → coach rows show no badge and the rank filter skips that athlete.

---

## UC33 — Daily missions

| | |
|---|---|
| **Actor** | System assigns; Athlete or Coach completes |
| **Precondition** | account active |
| **Postcondition** | up to 3 missions per local day per account; completed ones award XP through UC29 (athlete) or UC35 (coach) |

### Model

- **3 missions per day**, one `easy`, one `standard`, one `hard`, drawn from that role's catalogue at local midnight.
- XP by difficulty: athlete **30 / 60 / 120**; coach **25 / 50 / 100**.
- **Daily clear** — all 3 completed: athlete **+50**, coach **+40**.
- **Mission streak** — consecutive days cleared. 7 consecutive days: athlete +250, coach +200. The streak resets on the first day not cleared.
- Missions **expire at local midnight**. No carry-over, no backfill, no completing yesterday's.
- **One reroll per day**, on any single mission, before it has any progress.
- Progress is measured from the same Kafka events as everything else. Nothing is self-reported and no mission is marked done by tapping.

### Selection rules

1. Exclude missions completed in the last **3 days**.
2. Exclude missions the account cannot possibly do today: no step source connected → no step missions; coach with 0 athletes → no roster missions; athlete with no threshold set → no pace-relative missions; athlete with no prescribed session today → no session mission.
3. Weighted random within each difficulty from what is left.
4. Fewer than 3 eligible → assign what is eligible; the daily clear counts on those.

### Athlete catalogue

| Mission | Difficulty | Measured from |
|---|---|---|
| Walk 5,000 steps today | easy | step source (see Dependencies) |
| Walk 10,000 steps today | standard | step source |
| Log any activity today | easy | `activity.imported` |
| Start a run before 07:00 | easy | `started_at`, local |
| Finish a run with the watch field on | easy | `source = 'garmin_ciq'` |
| Run 5 km today | standard | `distance_meters` summed over the local day |
| Beat the average pace of your last run | standard | activity summary |
| Climb 200 m today | standard | elevation in the stream |
| Complete today's prescribed session | standard | `training.session.completed` |
| Run 10 km today | hard | daily distance |
| **Run at least 2 km 5 % faster than your threshold pace** | hard | rolling window over `activity_stream` |
| Hold an easy run below HR zone 3 for its whole duration | hard | HR stream + `max_heart_rate` |
| **Unlock a new place** | hard | UC34 |

### Coach catalogue

| Mission | Difficulty | Measured from |
|---|---|---|
| Send a workout to an assessoria | easy | UC14, `assessoriaIds` non-empty |
| Create a structured prescription | easy | UC13, `is_structured = true` |
| Open an assessoria dashboard | easy | UC21 |
| **Add 1 new athlete today** | standard | connection created — accepted request, either direction, or invite link |
| **Send 3 live commands today** | standard | UC28 `live_command` rows |
| Write feedback on 3 sessions | standard | UC22 |
| Set the threshold for an athlete who has none | standard | UC24 |
| Send a workout to 5 or more athletes | standard | UC14 `sessionsCreated ≥ 5` |
| Clear 3 items from the review queue | hard | UC19 `reviewed_at` |
| Follow an athlete live for 10 minutes | hard | UC28 live session duration |
| Check every athlete who missed a session this week | hard | one session detail opened per `missed` session |

### Main flow

1. At local midnight the assignment job writes 3 `mission_assignment` rows for the account, `progress = 0`.
2. First app open of the day shows the **Missions card** on Home: 3 rows, each with its target, its progress bar and its XP.
3. The account goes about its day. Every relevant Kafka event updates `progress` on any open mission it matches.
4. `progress` reaches `target` → `completed_at` set, XP awarded through UC29 / UC35, toast "Mission complete +60 XP", card row struck through.
5. Third completion → daily clear bonus awarded, `mission_streak` incremented, card shows "Day cleared".
6. Local midnight → unfinished missions expire, new 3 assigned.

### Alternative flows

- **Reroll.** Missions card → swap icon on a mission with `progress = 0` → `POST /me/missions/:id/reroll` → a new mission of the same difficulty, excluded from the same-day pool. One per day; the icon disappears afterwards.
- **One event completes two missions** (a 10 km run finishes "run 5 km" and "run 10 km"). Both complete; both award.
- **Mission becomes impossible mid-day** (the coach's only athlete disconnects; the athlete's session is deleted). The mission stays on the card, marked "no longer available", and does not block the daily clear — clearing counts the remaining ones.
- **Late upload.** An activity uploaded after midnight for a run done yesterday progresses **today's** missions if its `started_at` is today in local time, otherwise nothing. Missions never reopen.
- **Timezone change.** The day boundary follows the new timezone from the next assignment. The current day is neither shortened nor duplicated; no day is assigned twice.
- **Offline.** The card and progress come from the local cache with their timestamp. Completion is decided server-side only; nothing is awarded on the phone.
- **Backdated import.** Does not progress missions. Mission progress reads events whose `started_at` is the current local day.
- **Account inactive.** No assignment while inactive; nothing accrues.
- **Step source not connected.** Step missions are never assigned; the athlete sees a one-time card "connect steps to unlock step missions".

### Screens

| Where | Screen | Content |
|---|---|---|
| Athlete Home | Missions card | 3 rows: icon, text, progress bar, XP; "Day cleared" state; streak count |
| Coach Home | Missions card | same shape, coach catalogue |
| Both | Mission detail | what counts, what does not, progress so far, expiry countdown |
| Both | Completion toast | "+60 XP · 2 of 3 today" |
| Both | Missions history | last 30 days: assigned, completed, cleared days, streak |

### Dependencies

- **Steps are not in the data model today.** They come from the phone's health store (HealthKit on iOS, Health Connect on Android), read-only, with an explicit permission prompt, synced once an hour and on app foreground into `daily_steps (athlete_id, local_date, steps, source)`. The Garmin watch field does not send steps and the Garmin Health API is not available. Without this permission, step missions are simply never assigned.
- Elevation missions need elevation in the parsed stream (FIT and TCX carry it, most phone GPX does not).
- HR-zone missions need `max_heart_rate` set by the coach (UC24).

---

## UC34 — Unlock new places

| | |
|---|---|
| **Actor** | System (`gamification-svc`) reveals; Athlete views |
| **Trigger** | `activity.imported` with a GPS stream |
| **Precondition** | athlete account active; activity has usable GPS points |
| **Postcondition** | cities entered are unlocked; street pieces inside the vision radius are revealed; XP awarded through UC29 source 10; mission progress (UC33) evaluated |

### Model

The athlete owns one private **world map**, covered in fog. Running removes the fog. It works at two levels:

| Level | Unit | Unlocked by | Before unlock | After unlock |
|---|---|---|---|---|
| **City** | an OSM administrative boundary (municipality, `admin_level` 8, or the country's equivalent) | ≥ 200 m of valid track inside its boundary | gray area, **no name**, a "?" marker. A mystery | name and outline shown, city still gray inside |
| **Street** | a street piece: an OSM way cut into pieces of ≤ 50 m at import | a piece whose midpoint lies within the **vision radius** of the valid track | gray | full color |

Map states the athlete sees:

| State | Looks like |
|---|---|
| **Locked city** | flat gray, no label, "?" at its center. Neighbouring locked cities show only their borders |
| **Unlocked, unexplored** | city name, colored outline, "0 % explored"; streets inside still gray |
| **Unlocked, partly explored** | colored streets along and around the routes run; the rest gray; "37 % explored" |
| **Fully explored** | ≥ 90 % explored: whole city in color, gold outline (100 % is not reachable: private and closed streets) |

### Vision radius: "3 streets"

"3 streets around you" means a **distance**, not a hop count through the street graph. Hop counts reveal long ribbons down every cross street and behave differently in a grid than in a medieval center.

```
vision_radius(city) = clamp( 3 × median_block_length(city), 150 m, 400 m )
```

- `median_block_length` = median length of street pieces between two intersections in that city, computed once when the city's streets are loaded.
- A grid city with 100 m blocks gives **300 m**, which is the current street plus 3 parallel streets each side.
- A dense old center with 40 m blocks gives **150 m** (the floor), which is still 3 streets.
- A suburb with 200 m blocks gives **400 m** (the ceiling), so one run on a long road does not reveal a whole neighbourhood.

A street piece is revealed when its midpoint is within `vision_radius` of any valid track point. Long ways are already cut into ≤ 50 m pieces, so a highway reveals only the part near the runner.

### Valid track

A GPS point counts only if:

- horizontal accuracy ≤ 50 m (when the source reports it),
- speed over the surrounding 30 s ≤ **25 km/h**. Sections driven in a car or on a bus reveal nothing and unlock no city,
- the activity is not `manual` and not indoor (treadmill: no GPS stream, nothing to reveal).

### XP (UC29 source 10)

| Award | XP | Cap |
|---|---|---|
| **New city unlocked** | 300 XP | 5 cities per day |
| **New street revealed** | 2 XP per 100 m of newly revealed street length | 200 XP per activity |
| **City explored** milestones: 10 %, 25 %, 50 %, 75 %, 90 % | 200 · 500 · 1,000 · 2,000 · 4,000 XP | once per city per milestone |

`explored %` = revealed length ÷ runnable street length of the city. **Runnable** means footways, residential, tertiary and above, excluding motorways, trunk roads without sidewalks, private and `access=no` ways.

**Mission "Unlock a new place"** (athlete, `hard`) completes when the day's activities reveal **≥ 1 km of new street** or **unlock a city**.

### Main flow

1. `activity.imported` arrives with `athlete_id`, `activity_id`, `started_at` and a stream with lat/lng.
2. Idempotency: `processed_event` on `activity_id`. A replay reveals nothing twice.
3. The stream is filtered into the **valid track** (accuracy, speed, source).
4. **City resolution.** The valid track is intersected with `city.boundary` (spatial index). Any city whose streets are not loaded yet is loaded now (see Map data).
5. **City unlock.** For each city with ≥ 200 m of valid track inside and no `athlete_city` row, an `athlete_city` row is written (`unlocked_at`, `first_activity_id`) and 300 XP is awarded (cap 5/day).
6. **Street reveal.** For each of those cities: street pieces with midpoint within `vision_radius(city)` of the valid track that are not in `athlete_street` yet → inserted. Their summed length is the **newly revealed** length.
7. XP for newly revealed length (2 XP per 100 m, cap 200 per activity).
8. `athlete_city.revealed_m` is increased and `explored_pct` recomputed. Each milestone crossed awards once.
9. `athlete_city.fog_version` increments, so the app knows to refetch that city's fog.
10. Kafka `gamification.place.unlocked { athlete_id, activity_id, cities_unlocked[], revealed_m, cities_touched[] }`. UC33 consumes it for the mission.
11. Push, only for a new city: "New city unlocked: Campinas".

### Map data

- Source: **OpenStreetMap**. City boundaries for the whole world are loaded once (small: ~1 M polygons) and refreshed monthly.
- Streets are loaded **lazily per city**, the first time any athlete's track touches that city: extract the city's ways, cut them into ≤ 50 m pieces, compute `median_block_length`, `runnable_length_m` and `vision_radius`. The whole world's street graph is never loaded.
- While a city's streets are still loading, the activity is queued for that city and processed when loading finishes, in < 1 minute for a large city. The city unlock itself is not delayed.
- Monthly refresh: new streets are added gray; removed streets keep the athlete's reveal but leave the denominator. `explored_pct` never goes down for a milestone already awarded.

### Rendering

- Base map: MapLibre vector tiles, **full color**.
- The server returns a **fog polygon** per city per athlete: the city boundary minus the union of revealed pieces buffered by 25 m. The app draws it as a gray fill over the color map. Holes in the fog are what the athlete "sees".
- World zoom (< z9): each city is one shape. Locked: gray, no label, "?". Unlocked: name and a fill whose saturation follows `explored_pct`.
- City zoom (≥ z11): fog polygon over color streets, current activity's route drawn on top.
- Fog is cached per `(city_id, fog_version)`. Offline, the last cached fog is shown with its timestamp.

### Alternative flows

- **Backdated import.** Cities and streets **are** revealed (the map should reflect the athlete's history). XP is awarded only for activities with `started_at` in the last 90 days, the same boundary as constancy awards. No mission progress.
- **Activity superseded** (CIQ run replaced by its FIT upload). Reveal is not reversed. It is a union, so the replacement adds nothing new. XP entries are keyed by `activity_id` and reversed/re-awarded as in UC29.
- **Activity deleted.** XP is reversed; the reveal stays. Recomputing the map from the remaining activities is not worth the cost, and a street really was run.
- **Track crosses a border for < 200 m** (a road on the city line). No unlock; streets revealed inside that city are still stored and appear the day the city is unlocked.
- **GPS jump** (tunnel, lost signal). Points further than 200 m apart within 10 s are not joined; nothing between them is revealed.
- **Run in a car / on a bus** partway. Only the running sections count (speed filter).
- **Area with no OSM boundary** (sea, unorganised territory). Streets are revealed under a synthetic `city` per H3 resolution-5 cell, named after the nearest place node.
- **Two athletes run together.** Each reveals their own map. Maps are never shared or merged.

### Privacy

- The map is **visible only to the athlete**. Coaches never see it, not even with rank visible (UC32). No route exposes another person's map.
- The athlete can **share an unlocked city** to their groups (`SOCIAL.md` UC41): city name, explored %, date. The fog map and revealed streets are never shared.
- Activity routes are already private data; the map shows only what the athlete has already recorded.
- Deleting the account deletes `athlete_city` and `athlete_street`.

### Screens

| Where | Screen | Content |
|---|---|---|
| Athlete tab bar | **Map** | world map with fog; counters "12 cities · 148 km of streets"; tap a city → city card |
| Map | City card | name (or "?" when locked), `explored_pct` bar, next milestone, date unlocked, activities run there |
| Map | City zoom | fog over the city, routes run, "3 streets" radius shown as a soft halo along the last route |
| Activity detail | "Places" row | "+1 city · 2.4 km of new streets · +248 XP"; tap → the map centred on that activity |
| Unlock toast | after import | "Campinas unlocked" with the city silhouette going from gray to color |

### Schema additions

| Table | Notes |
|---|---|
| `city` | `id`, `osm_relation_id UNIQUE`, `name`, `country_code`, `boundary MULTIPOLYGON SRID 4326` + `SPATIAL INDEX`, `centroid POINT`, `streets_loaded_at NULL`, `runnable_length_m`, `median_block_m`, `vision_radius_m` |
| `street_piece` | `id`, `city_id`, `osm_way_id`, `geom LINESTRING SRID 4326` + `SPATIAL INDEX`, `midpoint POINT` + `SPATIAL INDEX`, `length_m`, `runnable BOOL` |
| `athlete_city` | `athlete_id`, `city_id`, `unlocked_at NULL` (NULL = streets stored but city not unlocked yet), `first_activity_id`, `revealed_m`, `explored_pct DECIMAL(5,2)`, `fog_version`, `UNIQUE (athlete_id, city_id)` |
| `athlete_street` | `athlete_id`, `street_piece_id`, `activity_id`, `revealed_at`, `PRIMARY KEY (athlete_id, street_piece_id)` |
| `city_milestone` | `athlete_id`, `city_id`, `pct ENUM('10','25','50','75','90')`, `awarded_at`, `UNIQUE (athlete_id, city_id, pct)` |

### Routes

| Method | Route | Who |
|---|---|---|
| GET | `/me/map/cities?bbox=` | athlete: cities in view, locked ones with no name |
| GET | `/me/map/cities/:id` | athlete: city card |
| GET | `/me/map/cities/:id/fog?v=` | athlete: fog polygon GeoJSON, `304` when `v` = current `fog_version` |
| GET | `/me/map/summary` | athlete: cities unlocked, km revealed |

### Tests

- Run 5 km in a 100 m grid city → vision radius 300 m; pieces 320 m from the track stay gray.
- Old center with 40 m blocks → radius 150 m; suburb with 200 m blocks → 400 m.
- Track touches a city for 150 m → no unlock, pieces stored with `unlocked_at NULL`; a later 300 m run there → unlock, those pieces appear at once.
- 12 km driven by car inside a run → nothing revealed on that section, no city unlocked from it.
- Same FIT twice → one reveal, one set of XP.
- A run revealing 15 km of streets → 200 XP (cap), not 300.
- Explored % 24 → 26 → 500 XP once; a monthly OSM refresh that drops it to 25.5 → no reversal, no second award.
- Backdated 2019 run through Paris → Paris unlocked, streets revealed, 0 XP, no mission progress.
- Locked city through `/me/map/cities` → `name: null`.
- Coach calling any `/me/map` route for an athlete → no such route exists.

---

## UC35 — Earn experience (coach)

| | |
|---|---|
| **Actor** | System (`gamification-svc`) |
| **Trigger** | connection, prescription, session, feedback, review, live and mission events |
| **Precondition** | coach account active |
| **Postcondition** | one `xp_entry` row per award; `coach_progress.xp` increased; UC30 evaluated |

### Sources

| # | Source | Award | Cap |
|---|---|---|---|
| 1 | **Athlete connected** — request accepted either direction, or invite link joined | 200 XP | 2 per day |
| 2 | **Workout sent** (UC14) | 25 XP per send, +2 XP per session created | 150 XP per day |
| 3 | **Session completed by one of your athletes** | 8 XP | 200 XP per day |
| 4 | **Feedback written** (UC22) | 15 XP | 10 per day |
| 5 | **Review queue item resolved** (UC19) — relinked or marked reviewed | 20 XP | 10 per day |
| 6 | **Live session followed** (UC28) — at least 5 minutes | 40 XP | 3 per day |
| 7 | **Live command seen on the watch** (UC28) | 5 XP | 60 XP per day |
| 8 | **Athlete threshold set** (UC24) — first time for that athlete | 60 XP | once per athlete |
| 9 | **Coaching week** — a week in which the coach sent at least one workout and wrote at least one feedback | 200 XP, +50 per consecutive qualifying week, capped at 500 | once per week |
| 10 | **Roster milestones** — 3rd, 5th, 10th, 25th, 50th, 100th athlete connected | 200 · 400 · 800 · 1,500 · 3,000 · 6,000 XP | once each, lifetime |
| 11 | **Your athlete crosses a tier** (UC30 on an athlete you are connected to) | 300 XP | once per athlete per tier |
| 12 | **Daily mission** | 25 / 50 / 100 XP by difficulty, +40 daily clear | see UC33 |

**Main flow**

1. The service consumes the event, resolves the coach or coaches it belongs to, and checks the day's cap for that source.
2. Under the cap → `xp_entry` written, `coach_progress.xp` incremented, UC30 runs.
3. At or over the cap → no entry; the mission card still shows progress if a mission tracks that action.
4. Weekly and roster rollups run hourly in the coach's timezone, keyed by period, so a rerun awards nothing twice.

**Alternative flows**

- **Athlete connected to two coaches.** Source 3 (session completed) awards the coach who prescribed that session. Source 11 (tier crossed) awards **every** connected coach. Sources 1 and 10 award the coach the connection was made with.
- **Athlete disconnects.** Nothing is reversed; XP already earned stands. The athlete no longer counts toward future roster milestones, and reconnecting the same athlete does **not** award source 1 or a milestone again.
- **Athlete reconnects after disconnecting.** Source 1 awards nothing (`UNIQUE` on the athlete id); the roster count treats them as the same person.
- **Session completed by a late upload** that flips `missed → completed`. Source 3 awards then, once.
- **Unplanned session.** No prescription, so source 3 awards nothing.
- **Coach follows an athlete live who is running someone else's prescription.** Sources 6 and 7 still award; source 3 does not.
- **Coach with no athletes.** Only sources 2 and 12 are reachable, and roster missions are never assigned.

---

## UC36 — View rank & progress (coach)

| | |
|---|---|
| **Actor** | Coach |
| **Precondition** | logged in |
| **Postcondition** | none (read-only) |

**Screens**

| Screen | Content |
|---|---|
| Coach Home — rank strip | tier badge, rank number, progress bar, XP remaining, mission streak |
| Rank detail | big badge, `xp`, `xp_to_next`, tier ladder, `rank_history` timeline |
| XP log | reverse-chronological entries: source, amount, date, the athlete or session it came from |
| Rank-up screen | shown once after UC30 |
| Coaching week card | workouts sent, feedback written, review items resolved, live minutes, this week against last |

**Main flow**

1. Coach Home → rank strip → `GET /me/rank` → `{ rank, tier, xp, xpToNext, weekStreak, missionStreak }`.
2. Tap → Rank detail.
3. "How XP works" → the UC35 source table with the coach's own totals and today's remaining caps per source.
4. XP log → `GET /me/xp?before=` → paged; tap an entry → the athlete or session it came from.

**Alternative flows**

- **Athletes never see a coach's rank.** No route exposes it to them.
- **No coach leaderboard.** Ranks are not compared across coaches anywhere in the app.
- **Offline.** Cached rank, XP and last log page with their timestamp.

---

## UC37 — Achievements

| | |
|---|---|
| **Actor** | System awards; Athlete and Coach view |
| **Trigger** | `activity.imported`, `activity.deleted`, `gamification.mission.completed`, `training.session.completed`, daily job (time since sign up), UC38 |
| **Precondition** | account active |
| **Postcondition** | `achievement_award` row per tier reached; `gamification.achievement.awarded`; shown on the profile |

### Model

- Achievements are **cosmetic records**. They award **no XP**: the thresholds that matter for XP (activity milestones) already pay through UC29 source 8, so an XP award here would count them twice.
- An insignia has **tiers I–VI** (VII for activities). The profile shows the **highest tier reached**; the detail shows every tier with its date.
- **Never revoked.** Like rank, a tier stays even if the count behind it later falls (activity deleted, superseded).
- Every account has them, from rank 1.

### Types

**1. Insignias**

| Insignia | Role | Counts | Tiers |
|---|---|---|---|
| **Veteran** | athlete, coach | time since sign up (`account.created_at`) | 1 month · 6 months · 1 year · 2 years · 3 years · 5 years |
| **Mission runner** | athlete, coach | missions completed (UC33), lifetime | 10 · 50 · 100 · 250 · 500 · 1,000 |
| **Activities** | athlete | activities logged (same count and thresholds as UC29 source 8; manual counts) | 10 · 25 · 50 · 100 · 250 · 500 · 1,000 |
| **Sessions delivered** | coach | sessions from the coach's prescriptions completed by their athletes (the coach's side of "activities completed") | 10 · 50 · 100 · 500 · 1,000 · 5,000 |

**2. Mastery: current and best of all time**

- Two slots on the profile, defined in **UC38**:
  - **Current Mastery:** this season's tier and MP. It resets to Starter (athlete) / Whistle (coach) on 1 Jan.
  - **Best of all time:** the highest tier any season ended on, with its year ("Champion · 2026"). It includes the current season, so it never falls.
- Each finished season also leaves a season badge on the profile ("2026 · Champion").

**3. Total hours of running** (athlete)

- A running total of `moving_seconds` from activities with a GPS or file source. **Manual activities are excluded**, because their time is typed, not recorded.
- Shown as a number ("212 h 40 min") with its own milestone insignia: **10 · 50 · 100 · 250 · 500 · 1,000 h**.
- Coach profile equivalent: **hours coached**, meaning moving time of sessions from the coach's prescriptions that athletes completed. Shown as a number, no insignia.

### Main flow

1. An event arrives (activity imported, mission completed, session completed) or the daily job runs in the account's timezone (Veteran).
2. The counter for that family is updated in `achievement_counter`.
3. Each threshold crossed and not yet in `achievement_award` → row written (`UNIQUE (subject_type, subject_id, achievement_key)`), `gamification.achievement.awarded`.
4. Push: "Insignia unlocked: Activities IV · 100 activities". The next app open shows an unlock card once.

### Where they show

| Profile | Seen by | Shows |
|---|---|---|
| **Athlete profile** | the athlete; their connected coaches (Athlete detail); members of their groups (`SOCIAL.md` UC43 profile card) | insignias, current / best Mastery, total hours |
| **Coach profile** | the coach; their connected athletes (My coaches → coach); any athlete who finds them by username (UC03 lookup card) | insignias, current / best Mastery, hours coached. **Never the coach's rank** (UC36 rule stands) |

### Screens

| Screen | Content |
|---|---|
| Profile → Achievements strip | one badge per insignia family at its highest tier; families not started are gray with "0 / 10" |
| Profile → Mastery card | current tier with MP bar, and best of all time, side by side (UC38) |
| Profile → Hours card | total hours; bar to the next hours milestone |
| All achievements | every family, every tier; reached ones with date, the next one with progress "37 / 50" |
| Unlock card | shown once after an award |

### Alternative flows

- **Backdated import.** Counts toward Activities and Hours; the insignia gets today's date, not the historical one.
- **Activity superseded** (CIQ → FIT). Counted once; the replacement does not add another activity or its hours again.
- **Activity deleted.** Counter decreases; tiers already reached stay.
- **Account inactive.** Veteran keeps counting from `created_at`; nothing else moves.
- **Several tiers at once** (a large import). One push "3 insignias unlocked", one unlock card listing them.

---

## UC38 — Mastery

| | |
|---|---|
| **Actor** | System (`gamification-svc`) scores; Athlete and Coach view |
| **Trigger** | `analysis.session.compared`, `training.session.missed`, `activity.imported` (unprescribed runs), mission daily clear (UC33), `training.session.feedback_written` (coach), season rollover |
| **Precondition** | account active |
| **Postcondition** | `mastery_entry` per scored event; `mastery_state` tier / MP updated; promotion event when a tier is crossed |

### Model

- Mastery is a **yearly season**, like ranked in League of Legends. A season is the calendar year in the account's timezone (1 Jan 00:00 → 31 Dec 23:59). On 1 Jan every account restarts at **tier 1, 0 MP**.
- Progress is in **Mastery Points (MP)**. Each tier needs a fixed amount of MP to promote; surplus carries into the next tier.
- **No demotion inside a season.** MP can fall to 0 inside the current tier but never below it, so a tier reached is kept until the reset. The loss still hurts, because the MP has to be earned back.
- Mastery is separate from rank (UC29–UC31): rank is lifetime XP and never resets, Mastery is **how well you train this year**. Neither feeds the other.
- **Most MP comes from prescribed sessions**, scored by how well they were done and how hard they were. Daily missions and unprescribed runs add a little.
- UC37 shows **current** Mastery (this season's tier) and **best of all time** (the highest tier any season ended on, current season included).

### Tiers

Same 10 tiers and MP for both roles, each with its own names. These are the names the rank tiers used to have (rank tiers are now numbered only), extended to 10. The ones marked *new* are proposals for review.

| # | Athlete | Coach | MP to promote | Season MP to reach | Workout tier = planned load of the session |
|---|---|---|---|---|---|
| 1 | **Starter** | **Whistle** | 100 | 0 | < 40 |
| 2 | **Runner** | **Guide** | 150 | 100 | 40–54 |
| 3 | **Strider** | **Trainer** *new* | 200 | 250 | 55–69 |
| 4 | **Pacer** *new* | **Mentor** | 300 | 450 | 70–84 |
| 5 | **Racer** | **Tactician** | 400 | 750 | 85–99 |
| 6 | **Contender** *new* | **Strategist** *new* | 500 | 1,150 | 100–119 |
| 7 | **Elite** | **Architect** | 650 | 1,650 | 120–139 |
| 8 | **Champion** *new* | **Visionary** *new* | 800 | 2,300 | 140–164 |
| 9 | **Icon** *new* | **Mastermind** *new* | 1,000 | 3,100 | 165–194 |
| 10 | **Legend** | **Maestro** | — | 4,100 | ≥ 195 |

**Workout tier** is how hard a prescribed session is. It is the session's **planned training load**: the load formula (`pace_v1`) applied to the prescribed blocks with the athlete's own threshold. Sixty minutes at threshold ≈ 100, and slower target paces give a lower load. So "a workout meant for a higher tier" means a session whose planned load falls in a higher band than the athlete's current tier. The bands are starting values, to be recalibrated with pilot data.

### Athlete: where MP comes from

| Source | MP | Cap |
|---|---|---|
| **Prescribed session** scored (below) | −12 … +40 | 2 sessions per day |
| **Session missed** (`training.session.missed`) | −5 | — |
| **Unprescribed run** (GPS or file, ≥ 20 min moving) | 4 × difficulty multiplier (workout tier from the run's actual load) | 1 per day, 3 per week; never negative |
| **Daily missions cleared** (all 3, UC33) | +2 | 1 per day |

Single missions give no MP; only a cleared day does. That caps missions at ~730 MP a year, which keeps activities the main path.

### Scoring a prescribed session

The comparison engine (UC17) already judges every repetition against the coach's tolerance (UC23). Mastery reuses that judgement.

**Step 1 — each repetition gets a score `s` from 0 to 1**

For repetition *i*, with target pace `T`, prescribed distance or duration `D` and the tolerance of its workout type:

```
pace_dev  = how far the achieved pace is off target, as a fraction of T
              work blocks:          only slower counts   (faster = 0)
              easy / recovery:      both directions count
dist_dev  = max(0, (D − achieved) / D)

p_pace    = pace_dev / pace tolerance (the slower or faster side, by direction)
p_dist    = dist_dev / distance tolerance
p         = max(p_pace, p_dist)        0 = on target, 1 = at the edge of the tolerance

s = 1                 if p = 0          target met, tolerance not needed
s = 1 − 0.5 × p       if 0 < p ≤ 1      inside the tolerance: from 1.0 down to 0.5
s = 0                 if p > 1          outside the tolerance, or repetition not run
```

**Step 2 — session quality `Q`**

```
Q = Σ (w_i × s_i) / Σ w_i        w_i = planned duration of repetition i
                                  (warm-up and cool-down weigh 0.5)
covered c = work actually run (distance or duration, as prescribed) / total planned work
```

**Step 3 — result and base MP**

| Result | Rule | Base MP |
|---|---|---|
| **Completed** | `c ≥ 0.9` and every repetition `p = 0` (`Q = 1`) | **+20** |
| **Almost completed** | `c ≥ 0.9`, some repetitions inside the tolerance, `Q ≥ 0.5` | **+20 × Q** → +10 … +19 |
| **Off target** | `c ≥ 0.9` but `Q < 0.5` (ran it all, mostly outside the tolerance) | **0** |
| **Gave up** | `c < 0.9`: the activity stopped with work repetitions not run | **−max(3, 12 × (1 − c))** → −3 … −12 |

**Step 4 — difficulty: harder or easier than your tier**

`d` = workout tier − athlete's current Mastery tier.

| `d` | ≤ −3 | −2 | −1 | 0 | +1 | +2 | ≥ +3 |
|---|---|---|---|---|---|---|---|
| **Gain multiplier** (completed, almost) | 0.4 | 0.6 | 0.8 | 1.0 | 1.3 | 1.6 | 2.0 |
| **Loss multiplier** (gave up) | 1.5 | 1.5 | 1.25 | 1.0 | 0.75 | 0.5 | 0.5 |

As in LoL, beating a harder opponent pays more and losing to an easier one costs more. Giving up on a session above your tier costs half; giving up on an easy one costs 1.5×.

```
session MP = round(base × multiplier)
```

### The cases

Athlete at **Racer (tier 5)**. Session: 6 × 1 km at 4:00 /km. Coach tolerance: pace 5 % slower, distance 5 %.

| Case | What happened | Q / c | Workout tier (`d`) | MP |
|---|---|---|---|---|
| **Completed** | every km at 4:00 or faster, full distance | Q = 1 | Racer (0) | 20 × 1.0 = **+20** |
| **Almost completed**, near the target | every km at 4:06 (2.5 % slower → p = 0.5 → s = 0.75) | Q = 0.75 | Racer (0) | 20 × 0.75 = **+15** |
| **Almost completed**, at the edge | every km at 4:12 (5 % slower → p = 1 → s = 0.5) | Q = 0.5 | Racer (0) | 20 × 0.5 = **+10** |
| **Almost completed**, mixed | 4 km at 4:00, 2 km at 4:06 | Q = (4 × 1 + 2 × 0.75) / 6 = 0.92 | Racer (0) | **+18** |
| Off target | every km at 4:20 (8 % slower, outside) | Q = 0 | Racer (0) | **0** |
| **Uncompleted (gave up)** | stopped after 3 of 6 km | c = 0.5 | Racer (0) | −max(3, 6) × 1.0 = **−6** |
| Gave up late | stopped after 5.5 of 6 km, first 5 on target | c = 0.92 → not a give-up; the unfinished 6th km has s = 0, so Q = 5 / 6 = 0.83 | Racer (0) | **+17** |
| **Harder, completed** | same quality, session planned at load 125 | Q = 1 | Elite (+2) | 20 × 1.6 = **+32** |
| **Harder, almost** | Q = 0.75 on an Elite session | Q = 0.75 | Elite (+2) | 15 × 1.6 = **+24** |
| **Harder, gave up** | stopped at half of an Elite session | c = 0.5 | Elite (+2) | −6 × 0.5 = **−3** |
| Much harder, completed | session at load 170 | Q = 1 | Icon (+4) | 20 × 2.0 = **+40** |
| **Easier, completed** | recovery run, load 60 | Q = 1 | Strider (−2) | 20 × 0.6 = **+12** |
| **Easier, gave up** | stopped at half of a Strider session | c = 0.5 | Strider (−2) | −6 × 1.5 = **−9** |
| Missed | never ran it | — | — | **−5** |
| Unprescribed run | 40 min, no prescription | — | — | **+4** |

What falls out of this:
- After the 1 Jan reset everyone is tier 1 (Starter), so their usual workouts count as harder (`d > 0`) and strong athletes climb fast. This replaces LoL's placement games.
- As the tier rises, the same workouts turn into `d < 0` and pay less. An athlete doing 4 Racer-load sessions a week, mostly completed or almost, with most days' missions cleared, ends the year around **Elite / Champion**. **Legend needs Icon- and Legend-load sessions done well.**
- A coach who prescribes slow target paces to make "completed" easy also lowers the planned load, so the session pays less.

### Coach Mastery

Coaches climb the same ladder from **how their athletes' sessions go**, not their own running.

| Source | MP | Cap |
|---|---|---|
| **Coaching day** — sessions of this coach's prescriptions scored that day | `round(12 × avgQ × min(1, n / 4))` | 1 per day (max 12) |
| **Feedback within 48 h** of the session (UC22) | +1 | 3 per day |
| **Daily missions cleared** | +2 | 1 per day |

- `avgQ` averages Q over that day's scored sessions, with gave-up and missed sessions counting as **Q = 0**. Sessions that could not be compared are left out.
- `n / 4`: four scored sessions a day earn the full amount, so roster size above that adds nothing and quality is what counts.
- **No difficulty multiplier for coaches.** Rewarding harder prescriptions would push coaches to overload athletes.

### Main flow

1. `analysis.session.compared` arrives with the repetition judgements, `c` and `confidence`.
2. The planned load gives the workout tier; `d` is taken from the athlete's tier **at the moment of scoring**.
3. Session MP is computed, capped (2 sessions/day) and written as a `mastery_entry` keyed by `session_id`. MP is floored at 0 within the tier.
4. MP reaches the tier's promotion value → tier + 1, surplus carried over, `gamification.mastery.promoted`, push "Promoted to Champion", promotion screen once.
5. The coach's coaching-day entry for that local date is recomputed with the new Q.
6. 1 Jan 00:00 local: `mastery_season` row closed with the final tier; the new season starts at tier 1 (Starter / Whistle), 0 MP.

### Alternative flows

- **Could not compare** (`confidence = 'low'` or `'not_compared'`, unstructured prescription). Scored like an unprescribed run (+4 × multiplier) until the coach relinks or reviews it (UC19). Relinking re-scores it: the old entry is reversed and a new one written.
- **Missed → completed by a late upload.** The −5 is reversed and the session is scored normally.
- **Prescription edited after scoring** (`stale`). Re-scored on the next comparison; the difference is a correction entry.
- **Activity superseded** (CIQ → FIT). Scored once, on the final comparison.
- **No threshold set.** No planned load, so the workout tier is taken as equal to the athlete's (`d = 0`).
- **Manual activity.** Cannot be compared and is not an unprescribed run either, because its time is typed: **0 MP**.
- **Two coaches.** Sessions from both count toward the athlete's Mastery; each coach's coaching-day MP uses only their own sessions.
- **Session scheduled on 31 Dec, result on 2 Jan.** Counts toward the season of `scheduled_date` if it arrives within 7 days; after that it scores nothing.
- **Backdated import.** No MP: Mastery reads only sessions and activities of the current season with results that arrive within 7 days.
- **Athlete without a coach.** Only unprescribed runs (max ~624 MP/year) and missions (max ~730) count, a ceiling of about **Contender**. Prescriptions are what open the upper tiers.

### Screens

| Screen | Content |
|---|---|
| Profile → Mastery card (UC37) | current tier emblem, MP bar to the next tier, best of all time, days left in the season |
| Mastery detail | the 10 tiers with the current one marked; MP log (session, result, Q, `d`, multiplier, MP); past seasons |
| Session detail (athlete) | "Mastery +24 · almost completed · harder (+2)" under the adherence block |
| Promotion screen | shown once: new emblem, the session that did it |
| Season end | 1 Jan, shown once: final tier of the year, badge added to the profile |

---

## Schema (`gamification_db`)

| Table | Notes |
|---|---|
| `athlete_progress` | `athlete_id PK`, `xp BIGINT`, `rank TINYINT`, `week_streak`, `mission_streak`, `last_award_at`, `show_to_coaches BOOL` |
| `coach_progress` | `coach_id PK`, `xp BIGINT`, `rank TINYINT`, `week_streak`, `mission_streak`, `last_award_at` |
| `xp_entry` | `id`, `subject_type ENUM('athlete','coach')`, `subject_id`, `source_kind`, `source_ref VARCHAR(128)`, `amount INT`, `awarded_at`, `UNIQUE (subject_type, subject_id, source_kind, source_ref)` |
| `personal_best` | `athlete_id`, `bucket ENUM('1k','5k','10k','21k','42k')`, `seconds`, `activity_id`, `set_at`, `last_award_at`, `UNIQUE (athlete_id, bucket)` |
| `rank_history` | `subject_type`, `subject_id`, `rank`, `reached_at`, `xp_at`, `UNIQUE (subject_type, subject_id, rank)` |
| `period_award` | `subject_type`, `subject_id`, `period_kind ENUM('week','month','year')`, `period_key` (`2026-W38`, `2026-09`, `2026`), `amount`, `UNIQUE (subject_type, subject_id, period_kind, period_key)` |
| `daily_cap` | `subject_type`, `subject_id`, `local_date`, `source_kind`, `used INT`, `UNIQUE (subject_type, subject_id, local_date, source_kind)` |
| `mission_definition` | `key PK`, `role ENUM('athlete','coach')`, `difficulty ENUM('easy','standard','hard')`, `target INT`, `unit`, `text`, `requires JSON` (the preconditions of the selection rules), `weight`, `active BOOL` — seeded, not user-editable |
| `mission_assignment` | `id`, `subject_type`, `subject_id`, `local_date`, `mission_key`, `difficulty`, `target`, `progress`, `completed_at NULL`, `expired BOOL`, `rerolled_from NULL`, `UNIQUE (subject_type, subject_id, local_date, mission_key)` |
| `mission_day` | `subject_type`, `subject_id`, `local_date`, `assigned`, `completed`, `cleared BOOL`, `reroll_used BOOL`, `UNIQUE (subject_type, subject_id, local_date)` |
| `daily_steps` | `athlete_id`, `local_date`, `steps`, `source ENUM('healthkit','health_connect')`, `UNIQUE (athlete_id, local_date)` |
| `achievement_definition` | `key PK` (`activities_iv`), `role ENUM('athlete','coach','both')`, `family ENUM('veteran','missions','activities','sessions_delivered','hours')`, `tier TINYINT`, `threshold INT`, `unit` — seeded |
| `achievement_counter` | `subject_type`, `subject_id`, `family`, `value BIGINT` (count, days or seconds), `UNIQUE (subject_type, subject_id, family)` |
| `achievement_award` | `subject_type`, `subject_id`, `achievement_key`, `awarded_at`, `value_at`, `seen_at NULL`, `UNIQUE (subject_type, subject_id, achievement_key)` |
| `mastery_state` | `subject_type`, `subject_id`, `season SMALLINT`, `tier TINYINT`, `mp_in_tier INT`, `mp_season INT`, `updated_at`, `UNIQUE (subject_type, subject_id, season)` |
| `mastery_entry` | `id`, `subject_type`, `subject_id`, `season`, `source_kind ENUM('session','missed','free_run','missions','coaching_day','feedback','correction')`, `source_ref`, `result ENUM('completed','almost','off_target','gave_up') NULL`, `q DECIMAL(4,3) NULL`, `covered DECIMAL(4,3) NULL`, `workout_tier NULL`, `d TINYINT NULL`, `multiplier DECIMAL(3,2) NULL`, `mp INT`, `created_at`, `UNIQUE (subject_type, subject_id, source_kind, source_ref)` |
| `mastery_season` | `subject_type`, `subject_id`, `season`, `final_tier`, `final_mp`, `closed_at`, `UNIQUE (subject_type, subject_id, season)` — best of all time = `MAX(final_tier)` with the current tier |
| `processed_event` | idempotency |

## Events

| Direction | Event |
|---|---|
| consumed | `identity.account.created`, `activity.imported`, `activity.deleted`, `activity.load_computed`, `identity.athlete.parameters_changed`, `identity.athlete.inactivated`, `identity.connection.created`, `identity.connection.ended`, `training.session.completed`, `training.session.missed`, `analysis.session.compared` (with per-repetition judgements and `covered`), `training.prescription.assigned`, `training.session.feedback_written`, `analysis.session.reviewed`, `live.session.started`, `live.command.seen` |
| produced | `gamification.xp.awarded { subject_type, subject_id, source_kind, amount, xp_total }`, `gamification.rank.changed { subject_type, subject_id, from, to, tier_changed }`, `gamification.mission.completed { subject_type, subject_id, mission_key, amount }`, `gamification.place.unlocked { athlete_id, activity_id, cities_unlocked[], revealed_m, cities_touched[] }`, `gamification.achievement.awarded { subject_type, subject_id, achievement_key }`, `gamification.mastery.scored { subject_type, subject_id, source_kind, source_ref, mp }`, `gamification.mastery.promoted { subject_type, subject_id, season, from, to }`, `gamification.mastery.season_closed { subject_type, subject_id, season, final_tier }` |

## Routes (gateway)

| Method | Route | Who |
|---|---|---|
| GET | `/me/rank` | athlete, coach — role decides which ladder answers |
| GET | `/me/xp?before=&limit=` | athlete, coach |
| GET | `/me/missions` | athlete, coach — today's 3 with progress |
| GET | `/me/missions/history?from&to` | athlete, coach |
| POST | `/me/missions/:id/reroll` | athlete, coach |
| PUT | `/me/rank/visibility` | athlete |
| PUT | `/me/steps` | athlete — hourly batch from the phone health store |
| GET | `/athletes/:id/rank` | coach (respects visibility) |
| GET | `/athletes?rankFrom=&rankTo=` | coach (list and UC14 filter) |
| GET | `/me/achievements` | athlete, coach — insignias, Mastery slots, hours |
| GET | `/me/mastery` | athlete, coach — season, tier, MP in tier, MP to promote, best of all time, past seasons |
| GET | `/me/mastery/log?before=` | athlete, coach — `mastery_entry` list |
| GET | `/sessions/:id/mastery` | athlete (own), coach (own session) — the scoring breakdown |
| GET | `/athletes/:id/achievements` | connected coach; group members through `SOCIAL.md` |
| GET | `/coaches/:id/achievements` | connected athlete, or any athlete from the UC03 lookup card |

## Rules

- Rank never decreases, for either role.
- XP is awarded from events, never from a request the app makes. A mission is never completed by tapping.
- Every award is idempotent on `(subject_type, subject_id, source_kind, source_ref)`; replaying Kafka from offset 0 reproduces the same totals.
- Constancy periods older than 90 days are never back-awarded; backdated imports never progress missions.
- Manual activities earn half distance XP and no personal best.
- Coach XP comes from coaching, never from the coach's own running. Athlete XP comes from running, never from being coached well.
- Coach roster sources are capped per day, not per athlete.
- Missions expire at local midnight and never reopen.
- Rank is never an input to adherence, training load, comparison or any dashboard number, for either role.
- The **only** ranking between people is the group rank inside an athlete group (`SOCIAL.md` UC43), visible to that group's members. No global or public leaderboard, and coaches are never ranked against each other.
- Achievements award no XP and are never revoked.
- Mastery resets every 1 Jan (account timezone); it never demotes inside a season; it is scored from events, and every entry is idempotent on `source_ref` so a re-score is a reversal plus a new entry.
- Mastery and rank never feed each other; neither feeds adherence, load or dashboards.
- Coaches get no difficulty multiplier; Mastery must never reward prescribing harder.

## Tests

- Same FIT uploaded twice → one set of awards.
- 40 km activity → 400 XP, not 400 + overflow.
- Five activities in one day → distance XP for all, activity-logged XP for two.
- Three consecutive qualifying weeks → 100 + 125 + 150 XP; a fourth week with 2 activities → 0 and streak reset.
- Threshold 4:40 → 4:30 → 250 XP; 4:30 → 4:40 → 0.
- XP total 203,940 → athlete rank 100; further XP → still 100.
- XP total 254,430 → coach rank 100.
- Backdated import of 300 activities → distance, activity and milestone XP; no week/month/year awards; no mission progress.
- CIQ run superseded by its FIT → negative entry, replacement entry, rank unchanged.
- A 10 km run completes both the 5 km and the 10 km mission.
- Reroll after progress → rejected; reroll twice in a day → rejected.
- Missions assigned at 00:00 in the account's timezone; a coach in Lisbon and an athlete in São Paulo get theirs at different UTC instants.
- Coach with 60 athletes and coach with 8 athletes, both sending one workout a day → daily XP differs by less than the cap allows.
- Athlete connected to two coaches crosses a tier → both coaches get source 11; only the prescribing coach gets source 3.
- Athlete disconnects and reconnects → no second source-1 award, no second roster milestone.
- 100th activity → XP from source 8 **and** Activities IV insignia; the insignia adds no XP.
- 100th activity deleted → count 99, Activities IV kept.
- 5 h of manual activities → Hours total unchanged.
- Account created 2025-09-23 → Veteran III (1 year) awarded on 2026-09-23 local date.
- Coach profile via lookup → achievements shown, rank absent.
- Mastery, Racer athlete, Racer session: every repetition on target → +20; every km 2.5 % slow with 5 % tolerance → +15; at 5 % → +10; at 8 % → 0.
- Gave up at 50 % → −6; same on an Elite session → −3; on a Strider session → −9.
- Elite session completed by a Racer athlete → +32.
- MP in tier 10, gave up for −12 → MP 0, tier kept.
- Session missed (−5), then a late upload flips it to completed → −5 reversed, session scored.
- 1 Jan 00:00 in São Paulo → season closed with final tier, new season Starter 0 MP; best of all time unchanged.
- Coach, 6 sessions in a day, Q = 1, 1, 1, 0.5, 0 (gave up), and one not compared → avgQ over 5 = 0.7 → round(12 × 0.7 × 1) = +8.
- Manual activity → 0 Mastery.
