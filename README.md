# Hearth

> A subscription library for **books, board games and video games**, with home delivery.
> Owners lend their items to a city hub and earn money each time an item is rented. Subscribers pay a monthly package to rent anything available.

**Status:** Phase 0 (setup) · **First hub:** Toronto · **Currency:** CAD · **Budget:** $0 (free tiers, Stripe in test mode)

Hearth is a **full-stack portfolio project**. It is meant to show that one developer can design, build, test, deploy and run a real two-sided product. When the fastest option and the option that shows more engineering skill differ, Hearth picks the second one. For example, it has its own REST API instead of a backend-as-a-service.

---

## Table of contents

1. [The problem](#the-problem)
2. [How Hearth works](#how-hearth-works)
3. [Users and apps](#users-and-apps)
4. [Item lifecycle](#item-lifecycle)
5. [Condition grading](#condition-grading)
6. [Subscription packages](#subscription-packages)
7. [Business rules](#business-rules)
8. [Trust, safety and privacy](#trust-safety-and-privacy)
9. [Scope](#scope)
10. [Architecture](#architecture)
11. [Tech stack (proposed)](#tech-stack-proposed)
12. [Data model](#data-model)
13. [Repository layout](#repository-layout)
14. [Getting started (local dev)](#getting-started-local-dev)
15. [Testing strategy](#testing-strategy)
16. [Roadmap](#roadmap)
17. [Success metrics](#success-metrics)
18. [Open questions](#open-questions)
19. [Glossary](#glossary)

---

## The problem

- Books and games sit unused on shelves.
- Buying a game you'll play twice is expensive.
- Libraries rarely stock board games or video games, and people don't have time to visit a store.
- People with full shelves have no safe, low-effort way to earn from them without meeting strangers.

## How Hearth works

Each city has **one physical hub**, a small warehouse with a front counter. Staff store, inspect, pack and ship every item there.

1. **Supply loop:** an owner registers items in the app and ships them to the hub, or drops them off. Staff inspect and grade each item and give it a product ID and barcode. The item then goes live in the catalog.
2. **Rental loop:** a subscriber picks items within their package's limits and chooses a delivery slot. A courier delivers the items. Later, a courier collects them, or the subscriber drops them at the hub.
3. **Earning:** the owner earns a **fixed amount per completed rental**, set per item type. Payouts run monthly.

Each item belongs either to a **person** (an owner) or to the **organizer**, which is Hearth's own stock and earns no owner payout.

## Users and apps

One account can be both a subscriber and an owner. Staff and admin are separate roles. All apps come from **one Next.js codebase** and talk to **one API**.

| App | Who | Form factor | Key screens |
|---|---|---|---|
| **Subscriber app** | Subscribers | Phone-first | Landing, sign-up/login, choose package, Stripe checkout, catalog (search + filters), item page, rental cart, delivery slot picker, orders & tracking, items I have, book a return, billing & package settings, notifications, profile & address |
| **Owner dashboard** | Owners | Phone-first | Register item (book / board game / video game), shipments & drop-off label, my items by product ID (renting / available), item detail with rental count, earnings & payouts, request return (after lock-in) |
| **Hub staff app** | Hub staff | Tablet-first | Intake queue, intake check (grade, photos, components, value, product ID label), shelf locations, pick lists, pack & dispatch, returns check-in, owner return requests, issue log |
| **Admin screens** | Organizer | Desktop | Packages (create / edit / retire), owner rates per item type, organizer stock, users & roles, charges & payouts, reports |

Every role has a **one-click guest demo login**, so a reviewer can try the app without signing up.

## Item lifecycle

Each box below is an `items.status` value.

```
 SUPPLY LOOP                                              RENTAL LOOP
 ┌────────────┐   ┌────────┐   ┌────────┐   ┌────────┐ subscriber ┌───────────┐
 │ Registered │──▶│ At hub │──▶│ Graded │──▶│  Live  │──orders──▶ │ Delivered │
 └────────────┘   └────────┘   └────────┘   └────────┘            └───────────┘
                                                ▲                       │ rental ends
                                  back in catalog                       ▼
                         ┌───────────────┐   ┌────────────┐      ┌───────────┐
                         │ Back to owner │◀──│ Checked in │◀─────│ Returned  │
                         │ (after 3+ mo) │   │ owner earns│      │courier/drop│
                         └───────────────┘   └────────────┘      └───────────┘
```

An item can go round the rental loop many times, and the owner earns at every check-in. After the **3-month lock-in**, an owner's return request takes the item out of the loop at its next check-in.

## Condition grading

The grade **never changes what the owner earns**. It only decides which copy is rented first: when several copies of a title are available, the newest, best-graded one goes out first.

| Grade | Books | Board games | Video games | Priority |
|---|---|---|---|---|
| New | Unread, perfect | Sealed or unplayed, all pieces | Sealed / no marks, case & inserts | 1st |
| Like new | No marks, spine uncracked | All pieces, box intact | No visible scratches, original case | 2nd |
| Good | Light wear, no writing | All pieces, light box wear | Light scratches, plays fully | 3rd |
| Fair | Visible wear, notes/creases | Minor wear, all key pieces | Visible scratches, plays fully | Last |
| Rejected | Missing pages, water damage | Not playable | Won't load, cracked, wrong region | Returned to owner |

Video game platforms: **PS5, Xbox Series X|S, Nintendo Switch**. The hub keeps one console of each so staff can test every disc at intake and at check-in.

**Check points:** intake → dispatch → check-in. Each check is saved as a `condition_report`, so every item has a full history.

## Subscription packages

Every subscription lasts **1 month and auto-renews** through Stripe Billing. Packages are **data, not code**: the admin can create any number of them, and the API enforces each package's limits on every order.

| Setting | Controls |
|---|---|
| Monthly price | Set by the admin (CAD) |
| Item types | Any mix of books / board games / video games, with an optional platform limit |
| Items at once | Overall or per type (e.g. 3 books + 1 video game) |
| Deliveries per month | Delivery/pickup runs included. Extra runs can be paid for |
| Premium items | Whether new releases and large board games are included |
| Status | Draft · Active · Retired (existing subscribers keep a retired package) |

Example packages (prices are set later):

| Package | Rents | Items at once | Deliveries / month |
|---|---|---|---|
| Reader | Books | 3 | 2 |
| Board Gamer | Board games | 2 | 2 |
| Console | Video games, one platform | 1 | 2 |
| Family | Books + board games | 5 | 4 |
| All Access | Everything incl. premium | 6 | Unlimited |

Subscribers can upgrade at any time (charged pro rata), downgrade at the next renewal, pause for up to 2 months, or cancel.

## Business rules

The API must enforce each rule below, and **each rule needs at least one automated test**. Each rule will become a GitHub issue labelled `rule`.

**Owners and items**
1. Owners register items online first. Staff can only receive registered items.
2. Staff grade each item at intake and assign a product ID and barcode. Rejected items go back to the owner.
3. An item goes live after intake, and its 3-month lock-in starts that day.
4. Owners can't request a return before the lock-in ends. After it ends, a requested item leaves the catalog at its next check-in and ships back.
5. Owners see only their own items: product ID, status, times rented, earnings, payouts and lock-in end date. They never see who rented an item.
6. Organizer-owned items earn no owner payout.

**Money**

7. Each completed rental creates one earnings entry at the fixed rate for that item type. The entry stores the rate that applied at the time.
8. Owner payouts run monthly, as the sum of unpaid earnings.
9. If an item is lost, the owner is paid the value recorded at intake.
10. **Balances are never stored.** They are always calculated from the ledger.

**Subscribers and rentals**

11. A subscription lasts 1 month and auto-renews until cancelled.
12. Subscribers can rent only the item types their package includes, within its item-at-once and delivery limits.
13. When several copies of a title are available, the copy with the best grade goes out first.
14. An item that is out can't be rented by anyone else.
15. Small damage and missing pieces are logged but not charged.
16. If an item isn't returned, its value is added to the subscriber's next bill, and they can't place new orders until it is settled.
17. Upgrades apply at once (charged pro rata). Downgrades apply at the next renewal.
18. A subscription can't end while items are still out.

## Trust, safety and privacy

- **Privacy by design:** owners and subscribers never meet or see each other's details. Only staff and couriers can see addresses, which are stored encrypted.
- **Verified accounts:** email verification, plus a valid card on file (through Stripe) for every subscriber.
- **Owner protection:** if an item is lost, the owner is paid the value recorded at intake.
- **Damage:** normal wear isn't charged. Staff log it, and repeat cases are flagged for the admin.
- **Data rights:** users can export or delete their data (GDPR-style). Payment data stays with Stripe.

## Scope

| First build (MVP) | Later | Out of scope |
|---|---|---|
| Sign-up/login with roles | Hobby kits (after design), rented through packages | Real payments (Stripe stays in test mode) |
| Owner item registration + read-only dashboard | More hubs and cities | Paid hosting or a paid domain |
| Hub intake: grade, product ID, barcode, shelve | Courier API + live tracking | Peer-to-peer meetups |
| Catalog with search & filters (type, genre, platform) | Recommendations, wishlists | Selling items |
| Admin-managed monthly packages | Push notifications, SMS | |
| Rental cart, delivery slots, order tracking | Expo mobile app | |
| Staff pick lists, dispatch, check-in | Analytics dashboard, ISBN scan | |
| Owner earnings ledger + monthly payouts (test mode) | | |
| Non-return charges on the next bill | | |
| Email + in-app notifications | | |
| Seeded demo data + guest logins per role | | |

## Architecture

```
┌──────────────┐      ┌──────────────────────┐      ┌─────────────────────────────────────┐
│    Users     │─────▶│  Next.js (Vercel)    │─────▶│  NestJS API (REST + Swagger, JWT)   │
│ subscribers  │      │  subscriber / owner  │      │  rentals, packages, ledger          │
│ owners, staff│      │  staff / admin apps  │      │  Socket.IO live updates             │
│ admin        │      │  barcode scanning    │      └──────┬───────────────────┬──────────┘
└──────────────┘      └──────────┬───────────┘             │                   │
                                 │ signed photo upload     ▼                   ▼
                                 │                 ┌──────────────┐    ┌──────────────┐
                                 └────────────────▶│ Image store  │    │ Redis        │
                                                   └──────────────┘    │ queue/cache  │
                                                   ┌──────────────┐    └──────▲───────┘
                                                   │ PostgreSQL   │           │
                                                   │ + full-text  │    ┌──────┴───────┐
                                                   └──────────────┘    │ Worker       │
                                                                       │ billing,     │
         Supporting: Email · Stripe (test) · Error monitoring · CI/CD  │ payouts      │
                                                                       └──────────────┘
```

- The browser uploads photos **straight to object storage** with a signed request, so large files never pass through the API.
- Front end and API live in **one monorepo** and share TypeScript types and validation schemas.
- Local development runs every dependency in Docker, at no cost.

## Tech stack (proposed)

> The stack below is the starting proposal from the planning docs. It is **still being reviewed** before Phase 0 is locked in. For every option with its pros, cons and a recommendation, see **`Hearth — Tech Stack Options and Decisions.pdf`**. Record final choices as ADRs in `docs/adr/`.

| Layer | Proposed choice |
|---|---|
| Language | TypeScript everywhere |
| Monorepo | Turborepo + pnpm workspaces |
| Front end | Next.js (App Router), React, Tailwind CSS, shadcn/ui, TanStack Query, React Hook Form + Zod, ZXing barcode scanner |
| API | NestJS, REST, OpenAPI/Swagger, validation pipes |
| Database | PostgreSQL + Prisma ORM & migrations + Postgres full-text search |
| Auth | Own JWT access + refresh tokens, argon2, Google OAuth, role-based guards |
| Real-time | Socket.IO gateway in NestJS |
| Background jobs | BullMQ + Redis |
| Files | Cloudinary (prod) / MinIO (local) |
| Payments | Stripe Billing, invoice items, Stripe Connect (all in test mode) |
| Email | Resend (prod) / Mailpit (local) |
| Monitoring | Sentry |
| Testing | Jest (unit), Supertest + Testcontainers (integration), Playwright (E2E), MSW (mock API) |
| DevOps | Docker, docker-compose, GitHub Actions, Terraform |
| Hosting ($0) | Vercel (web), Render (API + worker), Neon (Postgres), Upstash (Redis) |
| Mobile (stretch) | React Native + Expo |

**Free-tier trade-off:** the free API host sleeps when idle, so the first request after a quiet period can take 30 to 60 seconds. The demo page warns visitors about this. Free-tier limits change often, so check each provider's current terms before you set it up.

## Data model

The MVP has **22 PostgreSQL tables** in seven areas, and the hobby kit tables come later. Books, board games and video games share one `items` table with a `type` column. Money is always calculated from ledgers and never stored as a balance.

| Area | Tables | Purpose |
|---|---|---|
| People | `users`, `addresses` | Accounts with roles; encrypted delivery addresses |
| Hub | `hubs` | Hub location and opening hours |
| Inventory | `items`, `book_details`, `game_details`, `video_game_details`, `item_components`, `intake_batches`, `condition_reports`, `return_requests` | Each item, its details by type, components, the shipment it arrived in, its condition history, and owner returns |
| Packages | `packages`, `subscriptions` | Admin-defined packages; each subscriber's active one |
| Rentals | `rental_orders`, `rentals` | Delivery/pickup runs; each item's time with a subscriber |
| Money | `owner_rates`, `owner_earnings`, `payouts`, `subscriber_charges`, `lost_item_payouts` | Rates, earnings ledger, payouts, non-return charges, lost-item payments |
| System | `notifications`, `audit_log` | User messages; a record of every staff action |
| Later | `kit_products`, `kit_guides`, `guide_steps` | Hobby kits |

<details>
<summary>Key fields</summary>

| Table | Key fields |
|---|---|
| `users` | id, name, email, roles (subscriber / owner / staff / admin), stripe_customer_id, stripe_account_id |
| `addresses` | id, user_id, encrypted address, city |
| `hubs` | id, city, address, opening_hours |
| `items` | id (product ID), hub_id, type (book / board_game / video_game / kit), owner_type (person / organizer), owner_id, title, barcode, shelf_location, condition_grade, value, status, live_at, lock_in_until, rental_count |
| `book_details` / `game_details` / `video_game_details` | ISBN, author, genre / players, play time, age / platform, region, edition, age rating, premium flag |
| `item_components` | id, item_id, name, quantity |
| `intake_batches` | id, owner_id, shipping_method, tracking, received_at, status |
| `condition_reports` | id, item_id, stage (intake / dispatch / check-in), grade, missing_components, photos, note, staff_id |
| `return_requests` | id, item_id, owner_id, requested_at, status |
| `packages` | id, name, monthly_price, item_type_limits, items_at_once, deliveries_per_month, item_types, includes_premium, status, stripe_price_id |
| `subscriptions` | id, user_id, package_id, status (active / paused / past_due / cancelled), current_period_end, stripe_subscription_id |
| `rental_orders` | id, subscriber_id, kind (delivery / pickup / drop-off), slot, status (placed / packed / out / done), courier |
| `rentals` | id, item_id, subscriber_id, out_order_id, return_order_id, out_at, returned_at, status |
| `owner_rates` | id, item_type, amount_per_rental, active_from |
| `owner_earnings` | id, owner_id, item_id, rental_id, amount, created_at |
| `payouts` | id, owner_id, period, amount, stripe_transfer_id, status |
| `subscriber_charges` | id, subscriber_id, rental_id, reason, amount, stripe_invoice_item_id, status |
| `lost_item_payouts` | id, item_id, owner_id, amount, status |
| `notifications` | id, user_id, type, payload, read_at |
| `audit_log` | id, actor_id, action, entity, entity_id, at |

</details>

## Repository layout

```
hearth/
├── apps/
│   ├── web/              # Next.js: subscriber, owner, staff and admin apps (role-based routes)
│   └── api/              # NestJS: REST API, Socket.IO gateway, BullMQ worker
├── packages/
│   ├── shared/           # Shared types, Zod schemas, enums (item status, grades, roles)
│   ├── db/               # Prisma schema, migrations, seed script
│   └── config/           # Shared ESLint / TS / Prettier config
├── infra/
│   └── terraform/        # Free-tier infrastructure as code
├── docs/
│   ├── adr/              # Architecture Decision Records
│   └── api/              # OpenAPI contract
├── docker-compose.yml    # Postgres, Redis, MinIO, Mailpit
└── README.md
```

## Getting started (local dev)

> Nothing has been built yet, so these are the target steps. Update this section as the code arrives.

**Prerequisites:** Node.js (LTS), pnpm, Docker Desktop, Git.

```bash
# 1. Install dependencies
pnpm install

# 2. Start local services (Postgres, Redis, MinIO, Mailpit)
docker compose up -d

# 3. Configure environment
cp apps/api/.env.example apps/api/.env
cp apps/web/.env.example apps/web/.env.local

# 4. Create the database and seed the Toronto hub
pnpm db:migrate
pnpm db:seed

# 5. Run everything
pnpm dev
```

| Service | Local URL (planned) |
|---|---|
| Web app | http://localhost:3000 |
| API | http://localhost:4000 |
| Swagger docs | http://localhost:4000/docs |
| Mailpit inbox | http://localhost:8025 |
| MinIO console | http://localhost:9001 |

Stripe webhooks in local dev: `stripe listen --forward-to localhost:4000/webhooks/stripe`.

## Testing strategy

| Level | Tooling | What it covers |
|---|---|---|
| Unit | Jest / Vitest | State machines, allocation, package limits, lock-in, ledger math |
| Integration | Supertest + Testcontainers (real Postgres) | API endpoints, transactions, Stripe webhook handling |
| End-to-end | Playwright | Full loop: owner registers → staff intake → subscriber rents & returns → owner earns |
| Accessibility | axe + Lighthouse CI | WCAG AA, mobile Lighthouse 90+ |

CI runs all of these on every pull request.

## Roadmap

About 13 weeks at roughly 20 hours a week. A phase ends only when its **gate** is met.

| Phase | Weeks | Work | Gate |
|---|---|---|---|
| **0 · Setup** | 1 | Repo, CI, Prisma schema, seed data, OpenAPI draft, README | Runs locally |
| **1 · Design** | 2–3 | User journeys, wireframes, Figma design system, clickable prototype, 5-user test | All three journeys work in the prototype without help |
| **2 · Front end** | 4–6 | Three Next.js apps against a mock API (MSW), phone + tablet layouts | Every screen works on mock data |
| **3 · Back end** | 7–10 | NestJS API, auth, state machines, allocation, packages, Stripe, ledger, jobs, real-time, tests | Full loop passes E2E in CI |
| **4 · Cloud** | 11–12 | Free-tier deploy, Terraform, CI/CD, Sentry, Toronto seed (~300 items) | Live demo with guest logins |
| **5 · Showcase** | 13 | README, architecture diagram, ADRs, 2-min demo video, case study | Done |
| Later | — | Hobby kits, more hubs, courier API, Expo app | — |

### Phase 0 checklist

- [x] Pick the product name → **Hearth**
- [ ] Create free accounts (GitHub, hosting, database, Redis, image storage, Stripe test mode, email, monitoring, Figma)
- [ ] Create a public GitHub repo with the monorepo layout above
- [ ] Add ESLint, Prettier, conventional commits, and a GitHub Actions workflow that lints and tests every PR
- [ ] Add `docker-compose.yml` for Postgres, Redis, MinIO and Mailpit
- [ ] Write the Prisma schema for the 22 tables and run the first migration
- [ ] Write a seed script: Toronto hub, example packages, owner rates, sample items
- [ ] Turn each business rule into a GitHub issue labelled `rule`
- [ ] Draft the OpenAPI contract (auth, items, catalog, packages, rentals, staff actions)
- [ ] Finalise the tech stack and write the first ADRs

## Success metrics

| Measure | Target |
|---|---|
| Live demo | Guest login works in one click, no sign-up |
| Core loop | Owner registers → staff intake → subscriber rents & returns → owner sees earnings |
| API test coverage | ≥ 80%, run on every PR |
| E2E | Main flows pass in CI (Playwright) |
| Lighthouse (mobile) | 90+ performance and accessibility |
| API latency | Catalog search < 200 ms with 10,000 seeded items |
| Infrastructure | Rebuilt from scratch with one `terraform apply` |
| Docs | README, architecture diagram, Swagger, ADRs, demo video |

## Open questions

- [ ] Package prices: choose demo prices in CAD before seeding
- [ ] Owner rates: the fixed amount per rental for books, board games and video games
- [ ] Payout timing: which day of the month payouts run
- [ ] Final tech stack choices (see [Tech stack](#tech-stack-proposed))
- [ ] Hobby kits (parked): design contents, guides and restocking, then add kits to packages

**Decided:** one hub in Toronto with CAD prices · $0 budget · PS5 / Xbox Series X|S / Switch · subscriptions are always 1 month and auto-renew · stack is all TypeScript, with a separate API (no BaaS).

## Glossary

| Term | Meaning |
|---|---|
| **Hub** | The physical location where staff store, check and ship every item |
| **Owner** | A person who lends items to the hub and earns per rental |
| **Organizer** | Hearth itself, which can also own stock |
| **Subscriber** | A person paying a monthly package to rent items |
| **Package** | A monthly plan: price, item types and limits |
| **Product ID** | The ID and barcode given to each item at intake |
| **Lock-in** | The 3 months after going live when an owner can't take an item back |
| **Grade** | The condition rating that decides rental priority |
| **Ledger** | The list of earnings, payouts and charges that money totals are calculated from |
