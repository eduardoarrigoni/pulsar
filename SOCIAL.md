# Pulsar — Groups (athlete social)

**Scope:** athlete-made groups: sharing activity results, posts about places, sharing unlocked places, and a rank inside each group.
**Numbering continues** `GAMIFICATION.md` (UC29–UC38). This file: UC39–UC43.
**Owner:** `social-svc` (new). Owns `social_db`. Reads rank and places from Kafka events of `gamification-svc`; reads activity summaries from `activity.imported`.
**Not here:** coaching. Prescriptions, activities the coach monitors and performance dashboards belong to **assessorias** (`MOBILE.md` UC05, UC06).

| | Group | Assessoria |
|---|---|---|
| Created by | an **athlete** | a **coach** |
| Members | athletes only | athletes connected to that coach |
| What flows in it | results the athlete **chooses** to share, posts, unlocked places | prescriptions, every activity, performance |
| Prescriptions | **never** | yes |
| Rank | group rank by athlete rank (UC43) | none |
| Coach access | **none**. Coaches cannot create, join, find or read groups | owner coach only |

---

## Model

- A group has one **owner** (the creator) and **members**. Roles: `owner`, `member`. Max **200 members**.
- **Athletes only.** Coach accounts cannot create or join groups. A coach who runs uses their athlete account, as in `GAMIFICATION.md`.
- A group is **private**: its feed, member list and rank are visible to members only. It is found by exact **handle** (`@sunday-longrun`) or by invite link. There is no public directory and no search by partial name.
- Everything in a group is **opt-in per item**. Joining shares nothing automatically except the athlete's name, username, avatar, athlete rank and tier (needed for UC43).
- Nothing a member posts or shares in a group reaches any coach, not even the athlete's own coach.
- An athlete can be in any number of groups.

---

## UC39 — Create & manage group

| | |
|---|---|
| **Actor** | Athlete (owner) |
| **Precondition** | athlete account active |
| **Postcondition** | group created, edited, ownership transferred or archived |

**Main flow**

1. Groups tab → "+" → name, unique **handle**, optional description and cover image → `POST /groups` → group feed, owner is the only member.
2. Group → Settings (owner) → edit name, description, cover → `PATCH /groups/:id`.
3. Invite link → `GET /groups/:id/link` → `pulsar://group/<code>` + `https://` fallback, share sheet / QR.
4. Join requests → accept / decline (UC40 flow A).
5. Remove a member → member row → Remove → `DELETE /groups/:id/members/:athleteId`.

**Alternative flows**

- **Handle taken.** `409`; suggestions.
- **Rotate link.** `POST /groups/:id/link/rotate`; old link stops working; members stay.
- **Transfer ownership.** Settings → Transfer → pick a member → `POST /groups/:id/owner { athleteId }`.
- **Owner leaves.** Must transfer first; if they are the last member, leaving **archives** the group (read-only for 30 days, then deleted).
- **Archive.** Owner → Archive → feed read-only, no new posts or members; members can still leave.
- **200 members reached.** Link and requests return `409 "group full"`.

---

## UC40 — Join / leave group

| | |
|---|---|
| **Actor** | Athlete |
| **Precondition** | group active, not full |
| **Postcondition** | `group_member` row active, or `left_at` set |

**Flow A — request by handle**

1. Groups tab → Find → exact handle → card: name, cover, member count (no feed, no members) → Ask to join → `POST /groups/:id/join-requests`.
2. Owner gets a push and a badge → Accept → member; Decline → closed, the athlete can ask again after 7 days.

**Flow B — invite link**

1. Athlete opens the link → logged in: `POST /groups/join { code }`; if not: Sign up / Login, then the same call.
2. Joined immediately, no approval.

**Leave**

- Group → ⋯ → Leave → `POST /groups/:id/leave`. The athlete's posts and shares **stay** and show "former member". They can delete them before leaving.

**Alternative flows**

- **Already a member.** `200` no-op.
- **Removed by the owner.** Cannot rejoin by link for 30 days; can still ask (flow A).
- **Blocked member.** An athlete who blocks another member stops seeing that member's posts, shares and replies in every group. The rank still lists them.
- **Coach account opens a group link.** `403 "groups are for athletes"`.

---

## UC41 — Share to group (activity result, unlocked place)

| | |
|---|---|
| **Actor** | Athlete |
| **Precondition** | member of at least one group |
| **Postcondition** | one `group_share` per selected group |

A share is a **card** in the group feed, with an optional caption (≤ 280 characters). Two kinds.

### Activity result

What is shared, and nothing more:

| Always | Optional (off by default) | Never |
|---|---|---|
| date, type, distance, moving time, average pace, elevation gain | route map (first and last **200 m trimmed**), average HR | the prescription, coach feedback, adherence, training load, the HR stream, laps, anything from the coach |

**Main flow**

1. Activity detail → Share → pick groups (multi) → toggles for map / HR → caption → `POST /shares { kind: 'activity', activityId, groupIds[], includeMap, includeHr, caption }`.
2. `social-svc` snapshots the summary at share time. Later edits to the activity do not change the card.
3. Card appears in each group's feed; members can like and reply (UC42).

### Unlocked place

1. After UC34 unlocks a city or crosses an explored milestone, the unlock toast has **Share**.
2. Pick groups → `POST /shares { kind: 'place', cityId, event: 'unlocked' | 'explored_25' | 'explored_50' | 'explored_75' | 'explored_90', groupIds[], caption }`.
3. Card: city name, the athlete's `explored %`, date. **No fog map and no streets**: a fog map shows where someone lives and runs every day.

**Alternative flows**

- **Auto-share.** Group → My settings → "Auto-share new cities" (default **off**). When on, every city unlock posts a place card to that group. There is no auto-share for activities.
- **Manual activity.** Shareable, marked "manual", never with a map.
- **Activity deleted.** Its cards show "activity removed" and keep only the caption.
- **Unshare.** Card → Delete → removed from that group only.
- **Share to a group left meanwhile.** `403`; nothing posted.

---

## UC42 — Post about places

| | |
|---|---|
| **Actor** | Athlete |
| **Precondition** | member of the group |
| **Postcondition** | `group_post` created; replies and likes attached |

Posts work like X: short text, a reverse-chronological feed, replies and likes.

**Post**

- Text **≤ 280 characters**, up to **4 images**.
- Optional **place tag**:
  - a **city** the author has unlocked (UC34), or
  - a **spot**: a named point (e.g. "Parque Taquaral — water fountain") with its location rounded to ~100 m.
- Places are the point: the composer opens with "Where?" first, and the group feed can be filtered by city.

**Main flow**

1. Group feed → Compose → text, images, place → `POST /groups/:id/posts`.
2. Post appears at the top of the feed for all members; members with notifications on get a push.
3. Members **like** (`POST /posts/:id/like`) and **reply** (`POST /posts/:id/replies`). Replies are one level deep.
4. Tap a place tag → every post and share in this group tagged with that city or spot.

**Alternative flows**

- **Delete own post.** Removed with its replies. No editing, as on X; delete and post again.
- **Owner removes a post.** Removed; author sees "removed by the owner".
- **Report.** Post → Report → reason → goes to Pulsar moderation (outside the group); 3 reports from different members hide it until reviewed.
- **Tag a city not unlocked.** Not offered in the picker; `422` if forced.
- **Offline.** Post saved in the outbox and sent on reconnect, images included.
- **Notifications.** Per group: all posts / replies to me only (default) / off.

---

## UC43 — Group rank

| | |
|---|---|
| **Actor** | Athlete (viewer) |
| **Precondition** | member of the group |
| **Postcondition** | none (read-only) |

**The rank orders members by their athlete rank** (1–100, `GAMIFICATION.md` UC29–UC31), the one number every athlete has. The coach's level label (UC24) is per coach, private and unknown in a group, so it is not used.

| Order | Rule |
|---|---|
| 1st | athlete rank, highest first |
| tie | total XP, highest first |
| tie | earlier `rank_history.reached_at` for that rank |

**Main flow**

1. Group → Rank tab → `GET /groups/:id/rank` → position, avatar, name, tier badge, rank, XP to next rank; the viewer's own row pinned at the bottom if it is off screen.
2. Second view **"This week"**: XP earned this ISO week, highest first. Newcomers can lead here, while the all-time rank favours veterans.
3. Tap a member → their profile card: rank, tier, achievements (`GAMIFICATION.md` UC37).

**Alternative flows**

- **Rank-up of a member.** `gamification.rank.changed` → rank refreshes; a tier crossed posts an automatic card "Ana reached Tier 3" (member setting "announce my tier-ups", default on).
- **Member left.** Removed from the rank immediately.
- **Hide from rank.** Not available: being ranked is part of being in a group. Leaving the group is the way out.

---

## Screens

| Screen | Content |
|---|---|
| Groups tab | my groups with unread count; Find by handle; "+" |
| Group feed | posts and share cards, newest first; filter: all / activities / places / a city |
| Compose | "Where?" place picker, text, images |
| Post detail | post, likes, replies |
| Group rank | all-time / this week |
| Group info | description, members, invite link, my settings (notifications, auto-share cities) |
| Share sheet | from Activity detail and from the UC34 unlock toast |

## Schema (`social_db`)

| Table | Notes |
|---|---|
| `athlete_group` | `id`, `handle UNIQUE`, `name`, `description`, `cover_url`, `owner_id`, `invite_code UNIQUE`, `status ENUM('active','archived')`, `archived_at` |
| `group_member` | `group_id`, `athlete_id`, `role ENUM('owner','member')`, `joined_at`, `left_at NULL`, `removed BOOL`, `notify ENUM('all','replies','off')`, `auto_share_places BOOL`, `announce_tiers BOOL`, `UNIQUE (group_id, athlete_id, joined_at)` |
| `group_join_request` | `id`, `group_id`, `athlete_id`, `status ENUM('pending','accepted','declined')`, `created_at`, `decided_at` |
| `group_post` | `id`, `group_id`, `author_id`, `kind ENUM('post','share_activity','share_place','tier_up')`, `text VARCHAR(280)`, `images JSON`, `place_city_id NULL`, `place_spot_id NULL`, `payload JSON` (share snapshot), `deleted_at`, `hidden BOOL` |
| `group_reply` | `id`, `post_id`, `author_id`, `text VARCHAR(280)`, `deleted_at` |
| `post_like` | `post_id`, `athlete_id`, `PRIMARY KEY (post_id, athlete_id)` |
| `spot` | `id`, `name`, `city_id`, `location POINT` (rounded ~100 m), `created_by` |
| `member_rank_cache` | `athlete_id PK`, `rank`, `tier`, `xp`, `xp_week`, `reached_at` ← `gamification.*` events |
| `athlete_block` | `athlete_id`, `blocked_id`, `PRIMARY KEY (athlete_id, blocked_id)` |
| `report` | `id`, `post_id`, `reporter_id`, `reason`, `created_at`, `UNIQUE (post_id, reporter_id)` |

## Events

| Direction | Event |
|---|---|
| consumed | `activity.imported` (share summaries), `activity.deleted`, `gamification.place.unlocked`, `gamification.rank.changed`, `gamification.xp.awarded` (weekly XP), `identity.athlete.inactivated` |
| produced | `social.post.created { group_id, post_id, kind }` (push fan-out), `social.member.joined`, `social.member.left` |

## Routes (gateway, athlete only)

| Method | Route | Notes |
|---|---|---|
| POST / PATCH | `/groups`, `/groups/:id` | UC39 |
| GET | `/groups/lookup?handle=` | exact handle only |
| GET / POST | `/groups/:id/link`, `/groups/:id/link/rotate`, `/groups/join` | |
| POST | `/groups/:id/join-requests`, `.../:rid/accept`, `.../:rid/decline` | |
| POST / DELETE | `/groups/:id/leave`, `/groups/:id/members/:athleteId` | |
| GET | `/groups/:id/feed?before=&filter=` | |
| POST / DELETE | `/groups/:id/posts`, `/posts/:id`, `/posts/:id/replies`, `/posts/:id/like` | |
| POST | `/shares` | UC41 |
| GET | `/groups/:id/rank?view=all\|week` | UC43 |

Every `/groups*` route returns `403` to a coach token.

## Rules

- Groups are athlete-only. No coach ever reads, joins or receives anything from a group.
- Nothing is shared by joining except name, username, avatar, rank and tier. Every activity or place share is a separate choice.
- Route maps are trimmed 200 m at both ends; fog-of-war maps are never shared.
- Group rank is the athlete rank (1–100). The coach's level label is never shown in a group.
- Groups are private: no directory, no partial-name search.

## Tests

- Coach token on any `/groups*` route → `403`.
- Share an activity with the map on → first and last 200 m absent from the stored polyline.
- Share an activity → payload has no prescription, adherence, load or feedback fields.
- Two members with rank 24; the one with more XP ranks above.
- A member reaches rank Tier 3 with "announce" on → one tier-up card; off → none.
- Member leaves → their row disappears from the rank, their posts show "former member".
- Tag a city the author has not unlocked → `422`.
- Auto-share off (default) → a city unlock posts nothing.
