# Reverse proxy, API gateway, load balancer — the blunt version

Three names, a lot of overlap, and most of the confusion comes from vendors calling the same box three different things. Here's each one stripped down, then what Pulsar actually needs.

---

## 1. Reverse proxy

**What it is.** A server that sits in front of your real servers. The client talks to the proxy; the proxy talks to your app; the client never knows your app's address.

**Why it exists.**
- Your app shouldn't be the thing on the public internet. Node/NestJS is fine at running your code and bad at everything else: TLS handshakes, slow clients, malformed requests, 10 000 idle connections.
- One public IP/hostname, many apps behind it.
- Termination of HTTPS in one place, so your apps talk plain HTTP inside the network and you rotate one certificate, not five.

**How it works.**
```
client ──HTTPS──▶ reverse proxy (:443) ──HTTP──▶ app (:3000)
                     │
                     ├─ terminates TLS
                     ├─ adds X-Forwarded-For / X-Forwarded-Proto
                     ├─ buffers the request body (slow client doesn't hold your app's thread)
                     ├─ compresses the response
                     └─ serves static files itself, never bothers the app
```
It's a dumb pipe with a config file. It does not know what an "athlete" is. It routes on hostname and path, full stop.

**Common scenarios.**
- `api.pulsar.app` → gateway container; `pulsar.app` → static landing page. Same box, two `server` blocks.
- Hide the fact that the app is on port 3000 on an internal IP.
- Rate-limit by IP before the request costs you a database query.
- Block `/health` and `/metrics` from the outside.

**Products.** Nginx, Caddy (auto-TLS, minimal config), Traefik (auto-discovers Docker containers), HAProxy. Cloud: the "ingress" in Kubernetes is a reverse proxy.

**Rule of thumb.** You always have one. If you think you don't, your cloud provider is running it for you.

---

## 2. Load balancer

**What it is.** A reverse proxy that has **more than one** copy of the same app behind it and spreads requests across them.

**Why it exists.**
- One instance dies → the others keep serving. That's the real reason. Throughput is the second reason.
- Deploy without downtime: take instance A out, update it, put it back, repeat with B.

**How it works.**
```
                          ┌──▶ gateway #1
client ──▶ load balancer ─┼──▶ gateway #2
                          └──▶ gateway #3
                     │
                     ├─ health check every N s; stops sending to a dead instance
                     ├─ algorithm: round-robin (default), least-connections, IP-hash
                     └─ optional "sticky sessions" (same client → same instance) — avoid needing this
```

**The thing that bites people.** Load balancing only works if your instances are **interchangeable**. That means:
- No state in memory that the next request needs (sessions, "the file I just uploaded", in-progress jobs). Put it in MySQL/Redis/the broker.
- Uploaded files go to object storage, not the container's disk.
- Cron jobs must run on **one** instance only (or be idempotent). Three gateways = three `markMissed` runs unless you guard it. In Pulsar that's why cron lives in `training-svc` with a DB lock, not in the gateway.

**Layer 4 vs Layer 7.** L4 balances TCP connections — fast, blind, fine for databases and brokers. L7 reads HTTP — can route by path, inspect headers, do TLS. For an HTTP API you want L7; for MySQL/Kafka replicas you want L4.

**Common scenarios.**
- Two gateway replicas behind one LB so a deploy or a crash isn't an outage.
- Cloud: AWS ALB (L7) / NLB (L4), GCP HTTP(S) LB, Azure App Gateway. On a single VM: Nginx `upstream {}` block — it's the same Nginx, now with a list.

**Rule of thumb.** You don't need one until you have a second instance. You need a second instance the first time you deploy during business hours.

---

## 3. API gateway

**What it is.** A reverse proxy that **understands your API**. It knows about routes, clients, tokens, quotas, versions. It's the front door with a bouncer, not just a door.

**Why it exists.**
- Cross-cutting concerns in one place instead of in every service: authentication, rate limits per user, request logging, CORS, API keys, versioning (`/v1`, `/v2`).
- Mobile clients want one host and coarse endpoints; microservices expose many hosts and fine endpoints. Something has to translate.
- Protocol translation: HTTPS from the phone in, RabbitMQ/gRPC/whatever to the services out.

**How it works.**
```
client ──▶ API gateway ──▶ service A
               │      ├──▶ service B
               │      └──▶ service C
               │
               ├─ validates JWT, rejects before touching any service
               ├─ per-user rate limit (not per-IP — that's the proxy's job)
               ├─ routes /athletes/* → identity, /prescriptions/* → training
               ├─ aggregates: one screen = one request, gateway fans out to 3 services
               └─ transforms: internal ids → names, internal errors → public error envelope
```

**Two very different things wear this name:**

| | "Infra" API gateway | "Code" API gateway (BFF) |
|---|---|---|
| Examples | Kong, AWS API Gateway, Apigee, Tyk, KrakenD | Your own NestJS app that calls the services |
| Configured by | YAML / admin UI / plugins | code you write |
| Knows your domain? | no — routes, keys, quotas | yes — composes screens, resolves names, applies business-aware auth |
| Good at | auth offload, quotas, analytics, many teams, many clients | one client (your app), custom aggregation |
| Bad at | anything custom; plugin hell | you own the uptime and the boring parts (rate limits, retries) |

**Common scenarios.**
- Public API sold to third parties: infra gateway (keys, quotas, billing).
- One mobile app talking to your own microservices: **BFF** (Backend For Frontend) — a thin app whose only job is to serve that client well.
- Both: Kong in front of several BFFs (web BFF, mobile BFF, partner API).

**Rule of thumb.** If you're writing the client and the services, write the gateway too (BFF). Buy an infra gateway when someone *else* consumes your API.

---

## How they stack

They aren't alternatives. A normal production edge is all three, in this order:

```
internet
   │
   ▼
[ load balancer ]        cloud-managed or Nginx upstream — "which copy?"
   │
   ▼
[ reverse proxy ]        Nginx/Caddy/Traefik — TLS, compression, static, per-IP limits
   │                     (often the same process as the LB)
   ▼
[ API gateway ]          your NestJS BFF — JWT, per-user limits, route → RabbitMQ/Kafka
   │
   ▼
services
```

Small deployments collapse the top two into one Nginx/Caddy/Traefik process. Cloud deployments collapse them into a managed LB. Either way you still write the third one.

---

## What Pulsar needs

**Today (pilot, one assessoria, tens of athletes):**

| Layer | Decision | Why |
|---|---|---|
| Reverse proxy + LB | **Traefik** (or Caddy) as a Docker container in front of everything | auto-TLS from Let's Encrypt, routes by Docker labels so adding a second gateway replica is one `docker compose up --scale api-gateway=2`, and it *is* the load balancer once there are two. Zero config drift between "proxy" and "LB" because it's one thing |
| API gateway | **The NestJS `api-gateway` already in WORK-BLOCKS.md** — a BFF | one client (the RN app + the watch), heavy composition (names from identity, results from analysis), protocol translation to RabbitMQ. No off-the-shelf gateway does that; Kong would just be a second hop in front of code you still have to write |
| Infra API gateway (Kong etc.) | **No** | nobody external consumes the API. Revisit if a partner/API-for-clubs product appears |

**What the reverse proxy must do for Pulsar specifically:**
- Terminate TLS; the gateway sees `X-Forwarded-Proto: https` and trusts it (`app.set('trust proxy', 1)` in Nest/Express).
- `client_max_body_size` / equivalent ≥ 50 MB — bulk export ZIPs are big. Stream them to the gateway, don't buffer to RAM.
- Per-IP rate limit on `/auth/login` and `/ciq/*` (the watch has no JWT; a device-id is easy to spray).
- Only expose the gateway. Services, MySQL, Kafka, RabbitMQ and the RabbitMQ management UI are on the internal Docker network, never mapped to a public port.
- Long timeouts on `/me/imports` (upload), short everywhere else (the gateway's own RabbitMQ `send()` times out at 5 s anyway).

**What the gateway must do because there is a load balancer in front:**
- No in-memory state. Refresh tokens, invites, idempotency keys, watch bindings are already in `gateway_db` — keep it that way.
- Uploads go straight to object storage (or a shared volume in dev), never the container's disk.
- Push-notification consumer on Kafka uses one consumer group across replicas, so a `session.compared` event is turned into one push, not three.

**When to add more:**
- Second gateway replica: at first production deploy — it's free with Traefik and makes deploys painless.
- Service replicas: only the ones with work queues (`activity-svc` parsing, `analysis-svc` comparing). RabbitMQ is the load balancer for those — competing consumers, no proxy involved.
- Managed cloud LB instead of Traefik: when you leave a single VM. Traefik still stays as the ingress inside the cluster.
- Kong/APIM: when a third party gets an API key.

---

## One-line answers

- **Reverse proxy** — the bouncer's door. Always present. Nginx/Caddy/Traefik.
- **Load balancer** — the door, but with several identical rooms behind it. Needed the day you run two copies.
- **API gateway** — the bouncer who checks your ID and knows where each guest's table is. For Pulsar, that's the NestJS BFF you're already building; don't buy another one.
