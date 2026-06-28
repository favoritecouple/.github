# Favorite Couple — MVP Build Spec (for Claude Code)

Build spec for the MVP. Scope is deliberately narrow: **one country, one monthly competition.** The tiered ladder (regional/continental/global) is explicitly out of scope for v1 but the data model is designed so it can be added later without a rewrite.

---

## 1. Stack & Conventions

- **Framework:** Next.js (App Router, TypeScript, Server Components by default; Client Components only where interactivity is required).
- **DB:** PostgreSQL. Use **Prisma** or **Drizzle** as the ORM (pick one and keep it consistent — Drizzle recommended for type-safety + SQL transparency).
- **Cache / real-time / rate-limit:** Redis (counters, leaderboards via sorted sets, rate limiting, free-vote-per-day locks, background job queue).
- **UI:** shadcn/ui + Tailwind. Keep components in `components/ui` (shadcn) and `components/` (app-specific).
- **Auth:** Auth.js (NextAuth) with email + password and email verification, or Lucia. Email verification is **mandatory before voting** (core anti-fraud control).
- **Payments:** Stripe (Checkout for bundles, webhooks to credit accounts). EU VAT handling via Stripe Tax.
- **Hosting:** Self-hosted on Hetzner via Docker Compose (app + Postgres + Redis + reverse proxy with automatic TLS).
- **SEO:** Public pages are statically generated / ISR where possible, with full metadata, Open Graph images, structured data, and a sitemap.

**Code conventions for Claude Code:** strict TypeScript, server actions or route handlers for mutations, Zod for input validation on every mutation, no secrets in client code, all money stored as integer cents, all vote scoring computed server-side only.

---

## 2. MVP Scope

**In scope (v1):**
1. Auth + email verification (voters and couples).
2. Couple registration + public profile with story, photos/videos.
3. One monthly, country-level competition with a qualification (knockout) phase.
4. Voting engine: free daily vote, paid votes, 5× boosts.
5. Stripe payments for vote bundles and the purchasable boost.
6. Live leaderboard.
7. Daily couple posts.
8. Public SEO pages (couple profiles, competition page, leaderboard, landing).
9. Admin: moderation queue, content takedown, competition setup, prize config, manual winner payout.
10. Basic anti-fraud controls.

**Out of scope (v1) — design for, don't build:**
- Regional/continental/global tiers and auto-advancement.
- In-app native mobile apps.
- Automated payouts (manual for v1).
- Multi-country / multi-currency (single country, EUR only).
- Display advertising network (sponsor logo listings can be a simple admin-managed banner).

---

## 3. Data Model (Postgres)

Core tables (column lists illustrative, not exhaustive):

```
users
  id, email (unique), password_hash, email_verified_at,
  display_name, role (voter | couple_owner | admin),
  created_at, last_login_at, status (active | suspended)

couples
  id, owner_user_id (fk users), slug (unique, for SEO URL),
  title (e.g. "Anna & Marco"), bio, country_code,
  cover_media_id, status (active | hidden | banned),
  created_at

media
  id, couple_id (fk), url, type (image | video),
  width, height, moderation_status (pending | approved | rejected)

competitions
  id, level (country | regional | continental | global),  -- only "country" used in v1
  country_code, title, period_start, period_end,
  qualification_start, qualification_end,
  qualification_min_votes,                    -- knockout threshold
  prize_cents, prize_currency,                -- fixed, pre-announced
  prize_tiers_json,                           -- optional growing-prize milestones
  leaderboard_mode (weighted_votes | unique_voters),  -- see §5 decision
  status (draft | qualification | active | ended), created_at

competition_entries
  id, competition_id (fk), couple_id (fk),
  status (qualifying | qualified | eliminated | winner),
  qualified_at, auto_advanced (bool),         -- for future top-10 advancement
  unique(competition_id, couple_id)

votes
  id, competition_entry_id (fk), voter_user_id (fk),
  source (free_daily | paid),
  weight (int),                               -- 1 normally, ×5 during boost
  boost_applied (bool),
  created_at
  index(competition_entry_id), index(voter_user_id, created_at)

vote_credits
  user_id (fk), balance (int)                 -- purchased, unspent votes

boosts
  id, competition_entry_id (fk),
  type (free | paid), multiplier (default 5),
  window_start, window_end (24h), created_at
  -- one free + max one paid per entry per competition (enforced in app)

transactions
  id, user_id (fk), kind (vote_bundle | boost),
  amount_cents, currency, stripe_payment_intent,
  status (pending | succeeded | refunded),
  metadata_json, created_at

posts
  id, couple_id (fk), competition_id (nullable fk),
  text, media_id (nullable), created_at
  -- enforce max 1 per couple per day in app

sponsors
  id, name, logo_media_id, link_url,
  competition_id (nullable), placement (banner | logo_strip),
  active_from, active_to

moderation_reports
  id, target_type (couple | post | media | user),
  target_id, reporter_user_id (nullable), reason,
  status (open | actioned | dismissed), created_at, resolved_by

payouts
  id, competition_id (fk), couple_id (fk),
  amount_cents, status (pending | identity_verified | paid),
  method, notes, created_at, paid_at
```

---

## 4. Redis Usage

- **Free-vote lock:** key `freevote:{userId}:{competitionId}:{YYYY-MM-DD}` with TTL until end of day. Set on cast; existence blocks a second free vote that day.
- **Leaderboard:** sorted set `lb:{competitionId}`, member = `entryId`, score = total weighted votes. Increment on each vote; Postgres remains source of truth and is reconciled by a periodic job.
- **Rate limiting:** sliding-window counters per IP and per user on vote/auth/post endpoints.
- **Job queue:** BullMQ for async work (sending verification emails, OG image generation, leaderboard reconciliation, boost-window expiry).
- **Hot caches:** competition metadata, public profile data (short TTL, invalidate on edit).

---

## 5. Voting Engine (the core — get this right)

**Rules**
- A verified voter gets **one free vote per competition per day** (enforced via the Redis day-lock + a `votes` row with `source=free_daily`).
- A voter may cast **paid votes** by spending `vote_credits` (1 credit = 1 vote). Bundles purchased via Stripe top up the balance.
- Each casting writes a `votes` row with a computed `weight`.
- **Boost multiplier:** if the receiving entry has an active boost window at cast time, `weight = base_weight × 5` and `boost_applied = true`. Applies to both free and paid votes received during the window. ("5× all the votes they receive.")
- **Couples cannot vote.** A `couple_owner` cannot cast votes; block at the action level.
- All scoring is computed **server-side**; never trust a client-sent weight.

**Boosts**
- Each entry gets **one free 24h 5× window** per competition (couple chooses when to start it).
- Each entry may **purchase one additional 24h 5× window** per competition (Stripe). Price scales by competition level (config field; single price in v1).

**Qualification (knockout)**
- Any couple may enter; entry starts as `qualifying`.
- During `qualification_start → qualification_end` (e.g. 14 days) the entry must reach `qualification_min_votes` to flip to `qualified`.
- Entries not qualified by the deadline → `eliminated`.
- `auto_advanced` entries (future: top 10 of prior tier) skip qualification — stub the field now, no logic needed in v1.

**Leaderboard mode (decision flag — set per competition)**
- `weighted_votes` — score = sum of vote weights (your stated model; paid votes and boosts move the ranking).
- `unique_voters` — score = count of distinct verified voters; paid votes fund/grow the prize but do **not** move the ranking.
- Implement both behind `competitions.leaderboard_mode`. **Recommended default: `unique_voters`** — it removes the pay-to-win dynamic and the strongest incentive for couples to buy votes through proxy accounts, while paid votes still drive revenue and the prize. Flip to `weighted_votes` only if you deliberately want spending to decide the winner.

---

## 6. Payments (Stripe)

- **Vote bundles:** Stripe Checkout (e.g. 1 / 10 / 100 votes, lower per-vote price in bigger bundles). On `checkout.session.completed` webhook → increment `vote_credits.balance` inside a transaction, write `transactions` row.
- **Boost purchase:** Checkout for the single paid boost; webhook creates the `boosts` row and schedules expiry.
- **Idempotency:** key webhook handling on the Stripe event id; never double-credit.
- **Money:** integer cents, EUR only in v1, Stripe Tax for VAT.
- **Payouts:** manual in v1 — admin marks `payouts` paid after identity verification and bank transfer outside the app.

---

## 7. Public / SEO Pages

All public pages SSG/ISR with full metadata. These pages are the growth engine — each couple's page is shareable.

- `/` — landing: current competition, prize, top couples, CTA to vote/compete.
- `/c/[slug]` — **couple profile** (story, media, posts, vote button, share buttons). Highest-priority SEO page.
- `/competitions/[id]` — competition page: rules, prize (and growing-prize tiers), leaderboard.
- `/leaderboard/[competitionId]` — live ranking.
- **SEO requirements:** per-page `metadata` (title, description), Open Graph + Twitter cards, **dynamic OG image** per couple (generated via `@vercel/og` / Satori), `sitemap.xml`, `robots.txt`, JSON-LD structured data (`Person`/`Event` where sensible), canonical URLs, semantic HTML, fast LCP.
- **Share targets:** prefilled share links for Instagram/Facebook/TikTok/X/WhatsApp on every profile.

---

## 8. Admin & Moderation

- Admin-only area (`role = admin`).
- **Competition management:** create/configure a competition (dates, qualification threshold, fixed prize, growing-prize tiers, leaderboard mode), open/close, declare winner.
- **Moderation queue:** review reported couples/posts/media; approve/reject media; hide/ban couples; suspend users.
- **Takedown flow:** one-click hide of a couple or post (important for breakups/disputes) with audit log.
- **Sponsor management:** add sponsor logos/banners to a competition.
- **Payout management:** view winners, mark identity verified, mark paid.
- **Fraud review:** view flagged entries (see §9).

---

## 9. Anti-Fraud (build from day one)

- **Email verification required before any vote.**
- **One free vote per competition per day**, enforced server-side via Redis lock + DB constraint.
- **Rate limiting** per IP and per user on vote, signup, login, post endpoints.
- **Signal logging:** store IP, user-agent, and a device/session fingerprint on votes for later analysis (privacy-compliant; document in privacy policy).
- **Self-/proxy-voting detection:** background job flags entries whose voters cluster suspiciously (shared IP/device, signup bursts, voting only for one couple). Surface flags in the admin fraud review queue — no automated banning in v1.
- **`unique_voters` leaderboard mode** is itself the strongest structural anti-fraud measure (buying votes through proxies gains nothing on the ranking).

---

## 10. Deployment (Hetzner, self-hosted)

- **Docker Compose** services: `web` (Next.js, standalone output), `postgres`, `redis`, `proxy` (Caddy or Traefik for automatic Let's Encrypt TLS), `worker` (BullMQ jobs).
- **Next.js:** `output: "standalone"` for a small production image; multi-stage Dockerfile.
- **Reverse proxy:** Caddy recommended for zero-config HTTPS; route domain → `web:3000`.
- **Persistence:** named volumes for Postgres and Redis; mount media to a volume or use Hetzner Object Storage / S3-compatible bucket for uploads (recommended so the app stays stateless).
- **Backups:** nightly `pg_dump` to object storage; Redis is cache/derivable so lower priority (snapshot optional).
- **Env/secrets:** `.env` on the server (not committed); document required vars: `DATABASE_URL`, `REDIS_URL`, `NEXTAUTH_SECRET`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, mail provider keys, `S3_*`.
- **Migrations:** run on deploy (`drizzle-kit migrate` / `prisma migrate deploy`).
- **Observability:** structured logs, a basic uptime check, and Sentry (or similar) for errors.
- **CI/CD:** GitHub Actions → build image → push to registry → SSH deploy / `docker compose pull && up -d` on the Hetzner box.

---

## 11. Suggested Build Order (milestones)

1. **Foundation** — Next.js + TS + Tailwind + shadcn/ui scaffold, Drizzle + Postgres connection, Redis connection, Docker Compose for local dev.
2. **Auth** — signup/login, email verification, roles, session handling.
3. **Couples** — registration, profile editing, media upload (to S3-compatible storage), public profile page `/c/[slug]`.
4. **Competition + entries** — admin creates a competition; couples enter; qualification phase logic.
5. **Voting engine** — free daily vote, vote credits, casting, weight computation, Redis leaderboard, both leaderboard modes behind the flag.
6. **Payments** — Stripe Checkout for bundles + boost, webhooks, credit/boost provisioning.
7. **Boosts** — free + paid 5× windows, expiry job.
8. **Posts** — daily couple post (1/day cap).
9. **Public SEO pages** — landing, competition, leaderboard, OG images, sitemap, structured data.
10. **Admin & moderation** — queue, takedown, competition controls, payout management.
11. **Anti-fraud** — rate limiting, signal logging, flagging job.
12. **Deploy** — Hetzner Docker Compose, TLS, backups, CI/CD.

---

## 12. Non-Goals & Notes

- Prize is a **fixed amount set and announced before each round**, funded from revenue and/or sponsors — never a live percentage of incoming payments. The growing-prize feel is implemented as company-set milestone tiers (`prize_tiers_json`), not a redistribution of user funds.
- The **free daily vote** is the no-purchase entry route required by many EU promotional-competition rules — keep it always available and never gate it behind payment.
- This spec is structural/technical, not legal advice. Confirm prize-competition and payment/e-money rules for the launch country before going live, and reflect them in the published competition terms and privacy policy.
