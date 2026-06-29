# Favorite — MVP Build Spec (for Claude Code)

Build spec for the MVP. **Launch vertical = Pets, run as two competitions: Dogs and Cats**, in one country, monthly. The engine is a **reusable format** (register a contestant -> story -> free votes -> tiered ladder -> sponsors/ads), so it must be **category-aware from day one**: a vertical (pets, cars, gardens, couples) is configuration, not a code fork. The tiered ladder and additional verticals are out of scope for v1 but the data model supports them. **Core principle: money is fully decoupled from the contest outcome.** No paid votes, no paid boosts — voters never pay, and no payment affects who wins.

**Vertical roadmap (config, not rewrites):** Pets (Dogs + Cats) -> Cars -> Gardens -> Couples. Build the generic "contestant" model now; do not build later verticals until the pets loop is proven.

---

## 1. Stack & Conventions

- **Framework:** Next.js (App Router, TypeScript, Server Components by default; Client Components only where interactive).
- **DB:** PostgreSQL with **Drizzle** ORM (type-safe + transparent SQL).
- **Cache / real-time / rate-limit:** Redis (counters, leaderboards via sorted sets, rate limiting, free-vote-per-day locks, job queue).
- **UI:** shadcn/ui + Tailwind.
- **Auth:** Auth.js (NextAuth) or Lucia, email + password, **email verification mandatory before voting** (core anti-fraud control).
- **Payments:** Stripe — **only for contestant subscriptions.** No vote or boost purchases exist.
- **Ads:** Google AdSense (or equivalent) ad slots on public pages.
- **Hosting:** Self-hosted on Hetzner via Docker Compose (app + Postgres + Redis + reverse proxy with automatic TLS + worker).
- **SEO:** Public pages SSG/ISR with full metadata, OG images, structured data, sitemap.

**Conventions:** strict TypeScript, Zod validation on every mutation, no secrets client-side, money stored as integer cents, **all vote scoring computed server-side only.**

---

## 2. MVP Scope

**In scope (v1):**
1. **Category-aware engine** — a `verticals` config powering the launch vertical (Pets) with Dogs and Cats competitions; couples/cars/gardens addable later by config.
2. Auth + email verification (voters and contestant owners).
3. Contestant registration + public profile with story, photos/videos.
4. One country, monthly competitions (Dogs, Cats) with a qualification (knockout) phase.
5. Voting engine: **free daily vote** + **one free 5x boost per contestant per competition**.
6. Contestant subscriptions via Stripe (outcome-neutral perks).
7. Sponsor logo/ad listings (admin-managed) + AdSense slots.
8. Live leaderboard (per competition).
9. Contestant posts (daily; volume may be a subscription perk).
10. Public SEO pages incl. permanent evergreen story pages.
11. Admin: moderation, competition setup, prize config, sponsor management, winner/prize fulfilment.
12. Anti-fraud controls.

**Out of scope (v1) — design for, don't build:** higher tiers + auto-advancement; the Cars/Gardens/Couples verticals (engine supports them, but prove pets first); native mobile apps; cash payouts/payout rails; multi-country/multi-currency; any paid-vote or paid-boost feature (intentionally excluded, not deferred).

---

## 3. Data Model (Postgres)

```
verticals
  id, key (pets_dog | pets_cat | car | garden | couple | ...),
  display_name, contestant_noun (e.g. "dog", "car", "couple"),
  active (bool), config_json
  -- a vertical is configuration; launch with pets_dog + pets_cat

users
  id, email (unique), password_hash, email_verified_at,
  display_name, role (voter | contestant_owner | admin),
  created_at, last_login_at, status (active | suspended)

contestants
  id, owner_user_id (fk), vertical_id (fk), slug (unique, SEO URL),
  title, bio, country_code, cover_media_id,
  subscription_tier (none | premium), status (active | hidden | banned),
  created_at
  -- "contestant" is the generic entity: a dog, cat, car, garden, or couple

media
  id, contestant_id (fk), url, type (image | video),
  width, height, moderation_status (pending | approved | rejected)

competitions
  id, vertical_id (fk), level (country | regional | continental | global),  -- only "country" in v1
  country_code, title, period_start, period_end,
  qualification_start, qualification_end, qualification_min_votes,
  prize_description, prize_type (sponsor_inkind | giftcard | travel | cash),
  prize_value_cents, prize_funder (sponsor | company),
  prize_tiers_json,                       -- company-set growing-prize milestones
  status (draft | qualification | active | ended), created_at

competition_entries
  id, competition_id (fk), contestant_id (fk),
  status (qualifying | qualified | eliminated | winner),
  qualified_at, qualify_path (referral | votes | founder | auto_advance),
  unique(competition_id, contestant_id)

votes
  id, competition_entry_id (fk), voter_user_id (fk),
  weight (int),                           -- 1 normally, 5 during the entry's free boost window
  boost_applied (bool), created_at
  index(competition_entry_id), index(voter_user_id, created_at)
  -- all votes are free; no "source" payment distinction needed

boosts
  id, competition_entry_id (fk), multiplier (default 5),
  window_start, window_end (24h), created_at
  -- exactly ONE free boost per entry per competition, contestant-activated

subscriptions
  id, contestant_id (fk), stripe_subscription_id,
  status (active | canceled | past_due),
  current_period_end, created_at

transactions
  id, contestant_id (fk), kind (subscription),
  amount_cents, currency, stripe_invoice, status, created_at
  -- subscriptions are the only paid product

referrals
  id, referrer_contestant_id (fk), referred_contestant_id (fk),
  competition_id (fk), counted (bool), created_at
  -- counts only once referred contestant is verified + profile complete

posts
  id, contestant_id (fk), competition_id (nullable fk),
  text, media_id (nullable), created_at
  -- daily cap; higher for premium subscribers

sponsors
  id, name, logo_media_id, link_url, competition_id (nullable),
  placement (banner | logo_strip), funds_prize (bool),
  active_from, active_to

moderation_reports
  id, target_type, target_id, reporter_user_id (nullable),
  reason, status (open | actioned | dismissed), created_at, resolved_by

prize_fulfilments
  id, competition_id (fk), contestant_id (fk),
  type (sponsor_inkind | giftcard | travel | cash),
  status (pending | identity_verified | delivered),
  details, created_at, delivered_at

story_pages
  id, contestant_id (fk), slug (unique, permanent/canonical),
  competition_id (fk),                    -- the round it graduated from
  narrative (text, real story), published_at, updated_at
  -- created when a competition ends; URL never changes so link equity accumulates
```

---

## 4. Redis Usage

- **Free-vote lock:** `freevote:{userId}:{competitionId}:{YYYY-MM-DD}`, TTL to end of day. Blocks a second free vote that day.
- **Leaderboard:** sorted set `lb:{competitionId}`, member `entryId`, score = total weighted votes. Postgres is source of truth; periodic reconciliation job.
- **Rate limiting:** per-IP and per-user sliding windows on vote/auth/post endpoints.
- **Job queue:** BullMQ (verification emails, OG image generation, leaderboard reconciliation, boost-window expiry, referral validation).
- **Hot caches:** competition + public profile data, short TTL, invalidate on edit.

---

## 5. Voting Engine (the core)

**Rules**
- A verified voter gets **one free vote per competition per day** (Redis day-lock + `votes` row).
- **There are no paid votes.** Voters never pay; no vote credits exist.
- **Free boost:** each entry has **one free 24h 5x window per competition**, activated by the contestant owner at a time of their choosing. While active, votes that entry receives are recorded with `weight = 5`, else `weight = 1`.
- **Ranking** = sum of vote weights per entry (`lb:{competitionId}`). Since no money touches votes, there is no pay-to-win surface and no need for a separate "unique voters vs paid" mode — ranking is simply total weighted free votes.
- **Contestant owners cannot vote.** Block at the action level.
- All scoring server-side; never trust client-sent weight.

**Qualification (knockout)**
- Entry starts `qualifying`. A contestant qualifies via one of: **referral** (refer one verified, profile-complete contestant), **votes** (reach `qualification_min_votes` in the window), or **founder** (first launch cohort only, capped). `auto_advance` reserved for future top-10 tiers.
- The live round starts only after the qualification window closes; all qualified contestants enter together.

---

## 6. Payments & Ads

- **Stripe:** contestant subscriptions only (Checkout + customer portal; webhooks update `subscriptions`). EUR, Stripe Tax for VAT. Idempotent webhook handling keyed on Stripe event id.
- **No vote/boost purchases anywhere.**
- **Subscription perks (outcome-neutral by default):** verified badge, profile customization, voter analytics, priority support, cosmetic flair. **Visibility-affecting perks** (more posts/day, improved visibility, social features) are configurable and flagged — keep them off the competition ranking surface; gate behind a config flag pending legal review.
- **Ads:** AdSense (or equivalent) slots on public pages; admin-managed sponsor logo/banner placements per competition.
- **Prize fulfilment:** sponsor in-kind, gift card/voucher codes, or travel voucher — recorded in `prize_fulfilments`, delivered after winner identity verification. Cash is out of scope for v1 (would require payout rails).

---

## 7. Public / SEO Pages

- `/` — landing: current competition, prize, top contestants, CTA.
- `/c/[slug]` — contestant profile (story, media, posts, free vote button, share buttons). Top-priority SEO page.
- `/competitions/[id]` — rules, prize + growing-prize tiers, leaderboard.
- `/leaderboard/[competitionId]` — live ranking.
- `/story/[slug]` — **permanent evergreen "love story" page.** When a competition ends, an entry converts into a content-rich story page with a **canonical URL that never changes**, so link equity accumulates across rounds. This is the long-term organic-traffic asset.
- **SEO:** per-page metadata, OG + Twitter cards, dynamic per-contestant OG images (`@vercel/og`/Satori), `sitemap.xml`, `robots.txt`, JSON-LD, canonical URLs, fast LCP.
- **Share targets:** prefilled Instagram/Facebook/TikTok/X/WhatsApp links + referral link on every profile.

**Content-quality rule (critical for SEO):** story pages must be **genuinely content-rich**, not templated stubs. Prompt contestant owners to write a real narrative (minimum length / quality nudges), require real media. Mass-generated thin pages get deindexed and become a liability — the value is unique substance per page, not page count. SEO traffic compounds over 12–24 months, so build the fundamentals from day one; do not expect early organic traffic.

---

## 8. Admin & Moderation

- Competition management: create/configure (dates, threshold, fixed prize + type + funder, growing-prize tiers), open/close, declare winner.
- Moderation queue: review reported contestants/posts/media; approve/reject media; hide/ban; suspend. **Required to stay AdSense-eligible**, not only for safety.
- One-click takedown of a contestant/post (e.g. couples-vertical breakups/disputes) with audit log.
- Sponsor management: logos/banners, prize-funding flag.
- Prize fulfilment: verify winner identity, mark delivered.
- Fraud review queue (see Section 9).

---

## 9. Anti-Fraud (build from day one)

Even with no money on votes, fake accounts distort the contest, undermine prize integrity, and threaten AdSense eligibility.

- **Email verification required before any vote.**
- **One free vote per competition per day**, server-enforced (Redis + DB).
- **Rate limiting** per IP and per user on vote/signup/login/post.
- **Signal logging:** IP, user-agent, device/session fingerprint on votes (privacy-compliant; disclose in privacy policy).
- **Referral validation:** a referral counts only after the referred contestant is email-verified with a complete profile (story + at least 1 photo); flag shared device/IP clusters.
- **Clustering detection:** background job flags entries whose voters cluster suspiciously; surface in admin review (no auto-bans in v1).

---

## 10. Deployment (Hetzner, self-hosted)

- **Docker Compose:** `web` (Next.js standalone), `postgres`, `redis`, `proxy` (Caddy/Traefik, auto Let's Encrypt TLS), `worker` (BullMQ).
- **Next.js:** `output: "standalone"`, multi-stage Dockerfile.
- **Storage:** media to Hetzner Object Storage / S3-compatible bucket (keep app stateless).
- **Backups:** nightly `pg_dump` to object storage; Redis snapshots optional.
- **Secrets:** server `.env` (not committed): `DATABASE_URL`, `REDIS_URL`, `NEXTAUTH_SECRET`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, mail keys, `S3_*`, `ADSENSE_*`.
- **Migrations:** `drizzle-kit migrate` on deploy.
- **Observability:** structured logs, uptime check, Sentry.
- **CI/CD:** GitHub Actions -> build image -> registry -> SSH `docker compose pull && up -d`.

---

## 11. Suggested Build Order

1. **Foundation** — Next.js + TS + Tailwind + shadcn/ui, Drizzle + Postgres, Redis, local Docker Compose.
2. **Auth** — signup/login, email verification, roles.
3. **Contestants** — registration, profile, media upload to S3, public profile `/c/[slug]`.
4. **Competition + entries** — admin creates competition; contestants enter; qualification logic (referral + votes + founder).
5. **Voting engine** — free daily vote, free boost, weight + Redis leaderboard.
6. **Subscriptions** — Stripe Checkout + portal + webhooks; outcome-neutral perks.
7. **Posts** — daily post (cap; higher for premium).
8. **Public SEO pages** — landing, competition, leaderboard, OG images, sitemap, structured data, AdSense slots.
9. **Admin & moderation** — queue, takedown, competition controls, sponsor + prize fulfilment.
10. **Anti-fraud** — rate limiting, signal logging, referral validation, clustering job.
11. **Deploy** — Hetzner Docker Compose, TLS, backups, CI/CD.

---

## 12. Non-Goals & Notes

- Prize is a **fixed amount set and announced before each round**, funded by sponsor and/or company — never a percentage of revenue. Growing-prize feel = company-set milestone tiers.
- **No payment of any kind affects the contest outcome.** This is the central design rule; preserve it through every feature.
- The **free daily vote** is the no-purchase entry route — always free and ungated.
- **Visibility-affecting subscription perks** need legal review before being charged for; keep them ranking-neutral or behind a flag until cleared.
- Structural/technical spec, not legal advice. Confirm prize-competition rules for the launch country, and reflect them in published terms and the privacy policy, before going live.
