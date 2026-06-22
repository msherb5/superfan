# SuperFan

A one-stop sports hub for fans who are tired of switching between Twitter, Reddit, ESPN, YouTube, and their team's local beat reporter. SuperFan aggregates social content, live game data, player statistics, fantasy information, and community discussion into a single personalized feed — organized by the teams you follow.

**[Live Demo](https://superfan.vercel.app)** · **[API](https://superfan-api.onrender.com/health)**

> ⚠️ The API is hosted on Render's free tier and may take ~30 seconds to wake from idle on first request. See [Known Trade-offs](#known-trade-offs--what-changes-at-scale) for context.

---

## What It Is

SuperFan is a full-stack sports aggregation platform with five core features:

- **Live Feed** — A scrollable, filterable feed pulling articles, YouTube videos, and podcasts from national and local sports publications, aggregated per team and updated every 15 minutes via background workers
- **Team Hub** — Schedules, live scores, roster, player statistics, salary/cap data, and fantasy projections in one view
- **Player Profiles** — Per-player stats, contract details, injury history, and fantasy ownership/projection data sourced from Sportradar and the Sleeper API
- **Forum** — Reddit-style discussion boards scoped per team and globally, with nested comments, upvote/downvote scoring, post flair, and moderation controls
- **Messaging** — Private direct messages and group chats with real-time delivery, typing indicators, read receipts, and unread counts

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                         │
│                                                             │
│        Next.js 14 (App Router)        Expo React Native     │
│        Vercel · SSR + ISR             iOS + Android         │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS / WSS
┌──────────────────────────▼──────────────────────────────────┐
│                  MODULAR MONOLITH API                       │
│                  Fastify · TypeScript                       │
│                  Render (free tier)                         │
│                                                             │
│  ┌────────────┐ ┌────────────┐ ┌───────────┐ ┌──────────┐  │
│  │   Feed     │ │   Sports   │ │ Community │ │  Auth    │  │
│  │  Module    │ │   Module   │ │  Module   │ │  Module  │  │
│  │            │ │            │ │           │ │          │  │
│  │ BullMQ     │ │ Sportradar │ │ Forum     │ │ Supabase │  │
│  │ Workers    │ │ Sleeper    │ │ WebSocket │ │ RLS      │  │
│  └─────┬──────┘ └─────┬──────┘ └─────┬─────┘ └──────────┘  │
└────────┼──────────────┼──────────────┼───────────────────────┘
         │              │              │
┌────────▼──────────────▼──────────────▼───────────────────────┐
│                     DATA LAYER                               │
│                                                              │
│   Neon (PostgreSQL)        Upstash (Redis)                   │
│   Primary data store       Cache · Queue · Pub/Sub           │
│   Prisma ORM               BullMQ job persistence            │
└──────────────────────────────────────────────────────────────┘
         │
┌────────▼──────────────────────────────────────────────────────┐
│                  EXTERNAL DATA SOURCES                        │
│                                                               │
│  Sportradar API    Sleeper API    RSS Feeds    YouTube API     │
│  (scores, stats,   (fantasy       (articles,   (team and      │
│   player data)      projections)   podcasts)    media)        │
└───────────────────────────────────────────────────────────────┘
         │
┌────────▼──────────────────────────────────────────────────────┐
│                  AUTOMATION (GitHub Actions)                  │
│                                                               │
│  Feed cron (every 15 min)    Keep-alive ping (every 10 min)   │
│  CI/CD pipeline              Deployment on push to main       │
└───────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| **Web** | Next.js 14 (App Router) | SSR for team/player pages improves SEO and initial load; ISR keeps sports data fresh without full rebuilds; React Server Components reduce client bundle size |
| **Mobile** | Expo / React Native | Shared business logic and types with the web app via the monorepo; OTA updates without app store submissions |
| **API** | Fastify + TypeScript | Faster than Express with lower overhead; schema-based request validation built in; first-class TypeScript support |
| **ORM** | Prisma | Type-safe database access with migrations as code; generated types flow directly into the shared `@superfan/types` package |
| **Database** | Neon (serverless Postgres) | Production-grade Postgres that doesn't pause on inactivity (critical for a demo); supports branching for schema experiments |
| **Cache / Queue** | Upstash (serverless Redis) | Persisted Redis on a free tier; backs BullMQ for job queues, TTL-based API response caching, and WebSocket Pub/Sub fan-out |
| **Auth** | Supabase Auth | Handles OAuth (Google, Apple), JWT lifecycle, and session management; Row Level Security enforces data access at the database level |
| **Real-time** | Socket.io + Redis adapter | Bidirectional WebSocket connections for messaging and live feed updates; Redis adapter means multiple API instances share the same message bus |
| **Job Queue** | BullMQ | Persistent job queues with retry/backoff for feed aggregation workers; prevents external API rate limit violations during traffic spikes |
| **Monorepo** | Turborepo + npm workspaces | Incremental builds across packages; shared types, config, and DB client without publishing to npm |
| **Hosting** | Vercel (web) + Render (API) | Both have perpetual free tiers; zero-downtime deploys; clear migration path to Railway or AWS ECS when needed |
| **CI/CD** | GitHub Actions | Runs tests, lint, and deploys on push; also handles feed cron scheduling and API keep-alive pings |
| **Styling** | Tailwind CSS + shadcn/ui | Utility-first CSS with a component library that lives in the codebase rather than as a black-box dependency |
| **State** | Zustand + TanStack Query | Zustand for UI state (filters, modals); TanStack Query for server state with automatic caching, pagination, and background refetching |

---

## Key Technical Decisions

### 1. Modular Monolith over Microservices

The application is logically divided into four domains — feed, sports data, community, and auth — but they run as modules within a single Fastify process rather than as independent services.

Each module is self-contained with its own routes, service layer, and data access:

```
src/modules/
├── feed/
│   ├── feed.routes.ts
│   ├── feed.service.ts
│   └── feed.worker.ts
├── sports/
│   ├── sports.routes.ts
│   ├── sports.service.ts
│   └── clients/
│       ├── sportradar.ts
│       └── sleeper.ts
├── community/
│   ├── forum.routes.ts
│   ├── messaging.routes.ts
│   └── websocket.ts
└── auth/
    ├── auth.routes.ts
    └── middleware.ts
```

The boundaries are drawn along lines of independent scalability — in production, the feed module (high write volume from workers, high read volume from users during games) would be the first candidate for extraction into its own service. That extraction is a folder move and a network call, not a rewrite.

Running a true microservice mesh for a project with one developer and no sustained traffic would be operationally expensive for no architectural gain. The module boundaries make the intent clear without paying the cost of service-to-service networking at this stage.

### 2. Cursor-based Pagination over Offset

The feed, forum, and message history all use cursor-based pagination rather than `LIMIT / OFFSET`.

```typescript
// Offset pagination — breaks under concurrent writes
const posts = await prisma.forumPost.findMany({
  skip: page * limit,   // Row 50 shifts when new posts are inserted
  take: limit,
});

// Cursor pagination — stable regardless of writes
const posts = await prisma.forumPost.findMany({
  where: { createdAt: { lt: new Date(cursor) } },
  take: limit,
  orderBy: { createdAt: 'desc' },
});
```

Offset pagination produces duplicate or skipped items when new content is inserted between page loads — a near-constant occurrence in a live sports feed. Cursor pagination is stable because it anchors to a specific row rather than a numeric position in a result set. It also performs better at depth: `OFFSET 10000` requires the database to scan and discard 10,000 rows; a cursor query uses an index seek directly.

### 3. Denormalized Vote Scores on Forum Posts

The `score` column on `ForumPost` and `ForumComment` is denormalized — it stores the running vote tally rather than computing it with `SUM()` on the votes table at read time.

```prisma
model ForumPost {
  score    Int @default(0)   // denormalized running total
  votes    ForumVote[]       // source of truth for individual votes
}
```

A `SUM()` aggregation across potentially thousands of vote rows on every forum feed load would be expensive and hard to index. The denormalized score is a single integer read that supports efficient `ORDER BY score DESC` sorting. It's kept in sync by the vote mutation: when a user votes, the score is incremented/decremented atomically in the same transaction as the vote row upsert. The `@@unique([userId, postId])` constraint on the vote table prevents double-voting at the database level regardless of application logic.

### 4. Row Level Security for Private Data

Message and notification access is enforced at the database level via Supabase Row Level Security policies, not solely in application middleware.

```sql
-- Users can only read messages in chats they belong to
CREATE POLICY "chat_member_read"
ON "Message"
FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM "ChatMember"
    WHERE "ChatMember"."chatId" = "Message"."chatId"
    AND "ChatMember"."userId" = auth.uid()
  )
);

-- Users can only send messages as themselves
CREATE POLICY "message_insert_as_self"
ON "Message"
FOR INSERT
WITH CHECK (auth.uid() = "senderId");
```

Application-layer auth checks are necessary but not sufficient — a logic bug, a misconfigured route, or a compromised internal service could bypass them. RLS provides a second enforcement layer that cannot be bypassed from the application side. This pattern is particularly relevant for direct messages, where the consequence of a data leak is personal rather than just functional.

### 5. Redis Pub/Sub for WebSocket Fan-out

Socket.io is configured with a Redis adapter, meaning WebSocket events are broadcast through Redis rather than held in the memory of a single process.

```typescript
import { createAdapter } from '@socket.io/redis-adapter';

const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();

io.adapter(createAdapter(pubClient, subClient));
```

Without this, scaling the API to multiple instances would break messaging — a message sent to User A's connection on Instance 1 would never reach User B's connection on Instance 2. Redis Pub/Sub acts as the message bus between instances. The application code doesn't change when new instances are added; the adapter handles routing transparently. This is the same pattern used by Socket.io in production deployments.

### 6. Feed Deduplication via Unique Constraint

Feed items from external sources are stored with a composite unique constraint on `(type, externalId)`:

```prisma
model FeedItem {
  type       FeedItemType
  externalId String        // ID from source platform

  @@unique([type, externalId])
}
```

The feed aggregation workers run on a 15-minute cron cycle and will encounter the same content on every run. Rather than checking for existence before inserting, every write is an `upsert` — if the item already exists, the write is a no-op. This pattern is more reliable than a check-then-insert because it's atomic and doesn't create a race condition between concurrent workers.

### 7. GitHub Actions as Cron Infrastructure

Feed aggregation scheduling and API keep-alive pings run as GitHub Actions workflows rather than as always-on processes.

```yaml
# .github/workflows/feed-cron.yml
on:
  schedule:
    - cron: '*/15 * * * *'
jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - run: curl -X POST ${{ secrets.API_URL }}/internal/feed/trigger
               -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}"
```

An always-running scheduler process would either require a paid hosting tier or consume the single free Render service slot needed for the API. GitHub Actions provides 2,000 free minutes per month of cron-triggered compute. The trigger pattern also cleanly separates scheduling concerns from the API — the scheduler knows nothing about job implementation, and the API knows nothing about when it will be called.

---

## Database Schema Highlights

The full schema is in [`packages/db/prisma/schema.prisma`](packages/db/prisma/schema.prisma). A few design decisions worth calling out:

**Soft deletes on messages.** `Message.deletedAt` is nullable rather than hard-deleting rows. This preserves thread continuity (showing "this message was deleted" rather than breaking the conversation), keeps audit trails, and avoids cascading foreign key issues with any reactions or replies that referenced the message.

**Feed item team tagging via junction table.** A feed item can be tagged to multiple teams (`FeedItemTeam` junction). An ESPN article about a trade between the Cowboys and Patriots appears in both teams' feeds without duplicating the content row.

**Adjacency list for comment nesting.** Comments store a `parentId` reference to their parent comment. This supports arbitrary depth at the data layer while the application enforces a maximum depth of three levels — beyond which threads collapse into a "continue thread" link. A closure table would enable single-query subtree fetches but adds significant schema complexity for a depth limit that makes it unnecessary.

---

## Free Infrastructure Stack

This project runs entirely on free tiers with no credit card required.

| Service | Provider | Free Tier Used |
|---|---|---|
| Web hosting | Vercel | Unlimited hobby deployments |
| API hosting | Render | 750 hrs/month (one service) |
| PostgreSQL | Neon | 0.5 GB, no inactivity suspension |
| Redis | Upstash | 10,000 commands/day, persistent |
| Auth | Supabase | 50,000 monthly active users |
| Cron / CI | GitHub Actions | 2,000 minutes/month |
| Error tracking | Sentry | 5,000 errors/month |
| Sports data | Sportradar | Free trial key |
| Fantasy data | Sleeper API | Fully free, no key required |

**Monthly cost: $0**

---

## Known Trade-offs & What Changes at Scale

These are deliberate decisions made under the constraint of a zero-cost infrastructure budget. Each one has a documented path forward.

### Render cold starts (~30 seconds)
**Current:** Render's free tier suspends the API after 15 minutes of inactivity. A GitHub Actions workflow pings `/health` every 10 minutes to mitigate this during active hours, but a recruiter visiting at 3am may see a slow first load.

**At scale:** Move the API to Railway (starts at ~$5/mo) or AWS ECS Fargate for always-on compute. The API is fully containerized — this is a deploy target change, not a code change.

### Single database for all modules
**Current:** All modules share one Neon Postgres instance. This simplifies cross-domain queries and eliminates distributed transaction complexity at low traffic.

**At scale:** The community module (forum + messages) would be the first to extract, as it has the highest write volume and the most independent data model. Neon supports branching, so schema experimentation can happen without touching the production database.

### Mock data for Twitter/X and Instagram
**Current:** The Twitter and Instagram feed workers are architecturally complete but run against seeded mock data. The Twitter Basic API tier costs $100/month and Instagram Graph API requires a business account approval process — neither is practical for a portfolio project.

**At scale:** Swap the mock data seeder for real API credentials. The worker, queue, deduplication logic, and feed UI are unchanged. The only modification is the data source in the worker's fetch call.

### Upstash command limits
**Current:** Upstash's free tier provides 10,000 Redis commands per day. With 15-minute feed polling cycles and moderate forum activity, this is sufficient for demo traffic. Under real user load it would be exhausted quickly.

**At scale:** Upstash's Pay-as-you-go tier starts at $0.20 per 100,000 commands, or migrate to a dedicated Redis instance on Railway or AWS ElastiCache.

### No push notifications
**Current:** In-app notifications are fully implemented. Push notifications (mobile and web) are omitted — the setup overhead (OneSignal or APNs/FCM configuration) produces no demo value for a portfolio project.

**At scale:** Add OneSignal or direct APNs/FCM integration. The `Notification` table and notification-creation logic in the API are already in place; push delivery is an additive layer on top.

---

## Monorepo Structure

```
fanbase/
├── apps/
│   ├── web/                    # Next.js 14 web application
│   └── mobile/                 # Expo React Native app
├── services/
│   └── api/                    # Fastify modular monolith
│       └── src/
│           └── modules/
│               ├── feed/
│               ├── sports/
│               ├── community/
│               └── auth/
├── packages/
│   ├── types/                  # Shared TypeScript interfaces
│   ├── db/                     # Prisma schema + generated client
│   └── config/                 # Shared tsconfig, ESLint config
├── .github/
│   └── workflows/
│       ├── ci.yml              # Test + lint on PRs
│       ├── deploy.yml          # Deploy on push to main
│       ├── feed-cron.yml       # Feed aggregation trigger
│       └── keep-alive.yml      # Render ping
├── docker-compose.yml          # Local dev: Postgres + Redis
├── turbo.json
└── package.json
```

---

## Local Setup

### Prerequisites

- Node.js 20+
- Docker (for local Postgres and Redis)
- A Sportradar free trial API key
- A Supabase project (free)

### 1. Clone and install

```bash
git clone https://github.com/yourusername/superfan.git
cd superfan
npm install
```

### 2. Start local infrastructure

```bash
docker-compose up -d
# Starts Postgres on :5432 and Redis on :6379
```

### 3. Configure environment variables

```bash
cp .env.example .env
# Fill in values — see .env.example for descriptions of each variable
```

### 4. Set up the database

```bash
cd packages/db
npx prisma migrate dev
npx prisma db seed        # Seeds teams, sample players, and mock feed content
```

### 5. Run the development servers

```bash
# From the root — starts all apps and services in parallel
npm run dev
```

| Process | URL |
|---|---|
| Web app | http://localhost:3000 |
| API | http://localhost:3001 |
| Prisma Studio | http://localhost:5555 |

### 6. Seed mock feed data

Twitter and Instagram content uses mock data in development. To populate the feed:

```bash
cd services/api
npm run seed:feed
```

This inserts 50 mock feed items per seeded team, covering all five content types (article, YouTube, podcast, tweet, Instagram), so the feed UI is fully functional without API keys.

---

## Environment Variables

```bash
# Database
DATABASE_URL=postgresql://superfan:superfan_dev@localhost:5432/superfan

# Redis
REDIS_URL=redis://localhost:6379

# Supabase (auth only — database is Neon)
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Sports data
SPORTRADAR_API_KEY=          # Free trial at developer.sportradar.com
# TWITTER_BEARER_TOKEN=      # Optional — mock data used without this
# INSTAGRAM_ACCESS_TOKEN=    # Optional — mock data used without this
YOUTUBE_API_KEY=             # Free at console.cloud.google.com

# Internal
CRON_SECRET=                 # Random string — used to authenticate GitHub Actions cron calls
NEXT_PUBLIC_API_URL=http://localhost:3001
```

---

## API Reference

All endpoints require a valid Supabase JWT in the `Authorization: Bearer <token>` header, except `/health`.

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/health` | Service health check |
| `GET` | `/feed` | Paginated feed for followed teams. Params: `teamIds`, `types`, `cursor` |
| `GET` | `/teams` | All teams. Param: `sport` |
| `GET` | `/teams/:id` | Team detail with active roster |
| `GET` | `/teams/:id/scores` | Recent and upcoming games |
| `GET` | `/teams/:id/fantasy` | Fantasy stats and projections for team roster. Params: `season`, `week` |
| `GET` | `/players/:id` | Player profile |
| `GET` | `/players/:id/stats` | Player stats by season/week |
| `GET` | `/forum/boards` | All forum boards |
| `GET` | `/forum/boards/:id/posts` | Posts in a board. Params: `sort` (hot/new), `cursor` |
| `POST` | `/forum/boards/:id/posts` | Create a post |
| `GET` | `/forum/posts/:id` | Post detail with nested comment tree |
| `POST` | `/forum/posts/:id/vote` | Vote on a post. Body: `{ value: 1 \| -1 }` |
| `POST` | `/forum/posts/:id/comments` | Add a comment |
| `POST` | `/forum/comments/:id/vote` | Vote on a comment |
| `GET` | `/chats` | User's chat list with unread counts |
| `POST` | `/chats` | Create a DM or group chat |
| `GET` | `/chats/:id/messages` | Paginated message history |
| `GET` | `/notifications` | Unread notifications |
| `PATCH` | `/notifications/:id/read` | Mark notification as read |
| `GET` | `/search` | Cross-entity search. Params: `q`, `type` (players/teams/posts/all) |
| `POST` | `/internal/feed/trigger` | Trigger feed aggregation (cron only, requires `CRON_SECRET`) |

WebSocket events are documented in [`services/api/src/modules/community/WEBSOCKET.md`](services/api/src/modules/community/WEBSOCKET.md).

---

## Running Tests

```bash
# All packages
npm run test

# Specific package
npm run test --workspace=services/api

# With coverage
npm run test:coverage --workspace=services/api
```

---

## Deployment

The project deploys automatically on push to `main` via GitHub Actions.

- **Web** → Vercel (configured via `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID` secrets)
- **API** → Render (configured via `RENDER_DEPLOY_HOOK_URL` secret)

To deploy manually:

```bash
# Web
cd apps/web && npx vercel --prod

# API — push to main triggers Render auto-deploy,
# or use the Render dashboard for a manual deploy
```

Required GitHub Actions secrets:

```
VERCEL_TOKEN
VERCEL_ORG_ID
VERCEL_PROJECT_ID
RENDER_DEPLOY_HOOK_URL
API_URL                   # Production API URL for cron trigger
CRON_SECRET               # Must match CRON_SECRET in Render env vars
```
