# Ask.me

Intelligent, geolocation- and natural-language-based search platform for local businesses. The user asks in natural language (e.g. *"sushi open now near me that's pet-friendly"*) and gets back real establishments, filtered through a RAG pipeline — the LLM interprets intent and organizes the response, but **never invents data**: everything comes from the database.

The product is B2C on the acquisition side (free search) and B2B on monetization: business owners pay for a dashboard (`/dashboard`) to manage their own establishment page.

## Monorepo

```
ask.me/
├── backend/   # REST API — Node.js + TypeScript + Prisma + PostgreSQL/PostGIS
├── front/     # Mobile app — Flutter (iOS/Android)
├── web/       # Website — Next.js (React + TypeScript)
└── docs/      # Planning, agent prompt, static pages (privacy, how-to-use)
```

All three clients (`front/`, `web/`) consume the same REST API in `backend/`, under `/api/v1/...`. There's no BFF and no GraphQL — a deliberate decision (see [ADR-002](#adr-002-rest-instead-of-graphql)).

---

## Stack by layer

| Layer | Technology | Detail |
|---|---|---|
| Mobile | Flutter/Dart | Clean Architecture per feature (`data`/`presentation`), `provider` for state |
| Web | Next.js 16 (App Router) + React 19 + TypeScript | Tailwind v4, `react-markdown` for chat responses |
| Backend | Node.js + TypeScript + Express | `routes → controllers → services → repositories` layers |
| Database | PostgreSQL + PostGIS | Native geolocation (search radius) |
| ORM | Prisma | Migrations + typed client |
| LLM | Anthropic API (`@anthropic-ai/sdk`) | Interpretation/presentation layer, not a data source |
| Maps | Google Places API | Enrichment and automatic discovery of establishments |
| Auth | JWT + Google Sign-In (OAuth) | App's own token, issued after validating the Google `idToken` |
| Payments | Stripe (Checkout + Billing Portal) | Monthly B2B plan subscription |
| Deploy | Railway (backend/db) + Vercel (web) | Migrations automated in `preDeployCommand` |

---

## Backend (`backend/`)

REST API in Express, TypeScript, and Prisma, organized in layers:

```
src/
├── config/        # env.ts (variable validation), categories.ts
├── routes/        # admin, ask, auth, business, favorites, geocode, places, subscriptions
├── controllers/    # HTTP ↔ service translation
├── services/       # business logic (auth, subscription, discovery, nlSearch, maps...)
├── repositories/   # data access via Prisma
├── middleware/      # auth (JWT), errorHandler
└── db/             # seed, auxiliary migrations, place enrich/discover scripts
```

**Data model** (`prisma/schema.prisma`): `User`, `Subscription`, `Category`, `Place`, `OpeningHours`, `PlaceAttribute`, `PlaceEvent` (click analytics), `Favorite`, `BusinessClaim` (establishment claim by a business owner), `DiscoveredRegion` (tracking of already-scanned areas via Google Places).

**Main endpoints**:
- `GET /places` — traditional search (category, radius, "open now", pet-friendly)
- `POST /ask` — natural-language search (RAG); without `ANTHROPIC_API_KEY`, it automatically falls back to traditional search; **does not require login** (product decision, see commit history)
- `POST /auth/google` — exchanges the Google `idToken` for the app's own JWT
- `GET/POST/DELETE /favorites` — authenticated
- `POST /subscriptions/checkout|portal`, `GET /subscriptions/me` — Stripe
- `/business/*` — business owner dashboard: claim an establishment, edit photos/hours, manual review queue
- `/admin/*` — restricted to superadmin

**Running locally**:
```bash
cd backend
cp .env.example .env   # fill in DATABASE_URL, JWT_SECRET, etc.
docker compose up -d   # brings up Postgres + PostGIS via infra/Dockerfile
npm install
npm run db:migrate
npm run dev             # tsx watch src/server.ts
```

**Tests**: `npm test` (Jest, `--runInBand`), with `test:unit` and `test:integration` run separately.

---

## Mobile (`front/`)

Flutter app organized by feature, each with `data/` (repositories, DTOs) and `presentation/` (screens, view models via `provider`):

```
lib/
├── core/
│   ├── config/     # AppConfig — API base URL, public keys
│   ├── models/     # Place, Category, SearchFilters, User, SubscriptionStatus...
│   ├── network/    # ApiClient (Dio) — timeout, Authorization: Bearer injection
│   ├── services/   # geolocation, secure JWT storage
│   └── theme/      # palette #7C3AED / #FBF9FF
└── features/
    ├── search/         # traditional search, filters, list/detail
    ├── ai_search/       # conversational chat (RAG)
    ├── place_detail/
    ├── favorites/
    ├── account/         # Google login, session
    ├── business/        # business owner dashboard (B2B)
    └── admin/           # restricted area
```

**Authentication**: Google Sign-In → `idToken` sent to `POST /auth/google` → JWT stored in `flutter_secure_storage`.

**Running locally**:
```bash
cd front
flutter pub get
flutter run
```

---

## Web (`web/`)

Website in Next.js (App Router) — a rebuild of the Flutter app for the browser, consuming the **same backend**, without modifying the API (see [ADR-004](#adr-004-web-as-an-additional-client-not-a-replacement)).

```
src/
├── app/
│   ├── page.tsx (traditional search) │ ask/ (RAG chat) │ places/[id]/
│   ├── favorites/ │ login/ │ dashboard/ (business owner panel) │ admin/
│   ├── anuncie/ (B2B landing page) │ sobre/ (public landing page)
├── components/
└── lib/
    ├── api/       # central HTTP client, equivalent to Flutter's ApiClient
    ├── auth/      # session state
    └── favorites/
```

**Running locally**:
```bash
cd web
npm install
npm run dev   # NEXT_PUBLIC_API_BASE_URL pointing to the local backend
```

---

## Architecture decisions (ADRs)

A lightweight record of the decisions that shaped the project — the goal is for trade-offs not to get lost, even without a formal one-ADR-per-file process.

### ADR-001: Fixed stack — Flutter + Node/TypeScript + Postgres/PostGIS

**Context:** the founder keeps a full-time job during the early phase — the priority is low operational cost and low maintenance over "ideal" solutions that require full-time dedication.

**Decision:** Flutter (mobile, already mastered professionally) + Node.js/TypeScript (backend) + PostgreSQL/PostGIS (native geolocation) + LLM via a managed API (Anthropic) instead of self-hosted.

**Consequence:** high productivity for the MVP; revisit self-hosted LLM only after validating traction (Phase 6 of the roadmap in `docs/agent.md`).

### ADR-002: REST instead of GraphQL

**Context:** search filters (category, radius, "open now", pet-friendly) are known and limited in the MVP.

**Decision:** a simple REST API (`/places`, `/ask`, etc.) instead of GraphQL.

**Consequence:** less infrastructure to maintain; reconsider GraphQL only if the filters become combinatorially complex.

### ADR-003: LLM as an interpretation layer, never as a data source

**Context:** the most critical product risk is the LLM "hallucinating" establishments that don't exist.

**Decision:** the RAG pipeline (`nlSearchService`) always queries the database first; the LLM only reorganizes/explains real results. Without `ANTHROPIC_API_KEY`, `POST /ask` automatically falls back to traditional search (mandatory fallback, never an error).

**Consequence:** every LLM call has a deterministic fallback; this is a non-negotiable quality standard (see `docs/agent.md`).

### ADR-004: Web as an additional client, not a replacement

**Context:** the Flutter app (`front/`) already existed; there was a need to reach users without requiring an app install, and to gain SEO.

**Decision:** `web/` in Next.js consumes the existing backend without modifying it — the same API contract for both clients. Full feature mapping in `docs/web-plan.md`.

**Consequence:** zero business-logic duplication in the backend; the cost is maintaining two clients with equivalent UX.

### ADR-005: Monetization pivot — free B2C, paid B2B

**Context:** charging the end consumer (`R$39.90` → later `R$20`) showed high friction during validation.

**Decision:** AI search stays 100% open and free for the consumer; monetization shifts to a B2B dashboard (`/dashboard`) for business owners to manage their own page (photos, hours, contact channels), with a `R$99.90/month` plan.

**Consequence:** removed the paywall and subscription check on the consumer side; added an establishment-claim flow (`BusinessClaim`) with a manual review queue before automatic approval.

### ADR-006: Establishment discovery via Google Places, not manual curation

**Context:** manually populating the establishment database doesn't scale.

**Decision:** `discoveryService` scans regions via the Google Places API and persists them in `DiscoveredRegion` to avoid reprocessing the same area; `placeService`/`mapsService` enrich data (hours, attributes) on demand.

**Consequence:** dependency on Google's API cost/quota — a cost-vs-coverage trade-off taken on consciously (see `docs/plan.md`, Phase 0).

### ADR-007: Security treated as active debt, fixed incrementally

**Context:** recent commits fixed: stored XSS via a `javascript:` URL (menu/JSON-LD on the site), open CORS with no allowlist, missing security headers, an internal Vercel URL leaking in sitemap/robots/Stripe, and `.env` leaking in integration tests.

**Decision:** every security finding becomes a dedicated, immediate commit, not a backlog item; `helmet`, `express-rate-limit`, and `ALLOWED_ORIGIN` (restricted CORS) have been standard since the API's inception.

**Consequence:** attack surface reviewed continuously; any new route/client must keep CORS restricted and must never reintroduce secrets into versioned code.

---

## Additional documentation

- [`docs/plan.md`](docs/plan.md) — phased implementation roadmap (foundation → backend → mobile → RAG → MVP → scale)
- [`docs/web-plan.md`](docs/web-plan.md) — Flutter → Next.js conversion plan, feature mapping
- [`docs/agent.md`](docs/agent.md) — reference prompt for AI agents acting as tech lead on the project (quality standards, business constraints)
- [`docs/privacy-policy.html`](docs/privacy-policy.html), [`docs/how-to-use.html`](docs/how-to-use.html) — public static pages
