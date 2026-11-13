# OpenReply — deployment checklist (brva fork, AUTO mode)

> Personal fork of `diwenne/openreply` (MIT) for `@tir_vision` and
> `@brva.rashidpour`. Instagram comment replies and DMs send **automatically,
> without per-message approval** (owner's explicit decision, 2026-11-13).
> Post/reel publishing is NOT part of this — publishing still needs the
> owner's approval.
>
> **No Zernio.** Direct Meta app only.
>
> **Secrets live only in host env settings / the Secure Vault — never in
> this repo, never in chat.** This file contains variable *names* and
> generation *commands* only.

## Architecture — $0/month

| Piece | Service | Cost | Notes |
|---|---|---|---|
| Web app (Next.js 16) | Vercel Hobby | Free | Public URL (`*.vercel.app`) used for webhooks + OAuth callback |
| Worker (always-on) | Botkeep Node.js slot | Free | 1 of the 2 free slots (slot 1 = auction radar). Start: `npm run worker` |
| PostgreSQL | Neon free | Free | ~0.5 GB |
| Redis (TCP) | Redis Cloud Essentials | Free | 30 MB, native TCP (BullMQ needs TCP, not HTTP-only Redis) |
| Login email | Resend free (3k/mo) **or** own SMTP | Free | Own SMTP = SendPulse on `brvarashidpor@brva.my.id` via `EMAIL_SERVER`; then no Resend key needed |
| Instagram API | Own Meta app | Free | No App Review for the owner's own accounts (added as testers) |

Vercel's `vercel-build` script already runs migrations:
`prisma generate && prisma migrate deploy && next build`.
Vercel crons (daily token refresh 05:00) are in `vercel.json`.

**What cannot run here:** Cloudflare Workers/Pages cannot run any piece
(both processes need long-lived Node + TCP to Postgres/Redis). Railway no
longer has a free tier; Render free sleeps (kills the worker); Fly's free
allowance is gone. **Paid fallback if the Botkeep worker fails:
Railway worker + Postgres + Redis, ~$5–10/month.**

## 0. Prerequisite — the fork (owner action, GitHub web UI)

The Muse GitHub App cannot fork or create repositories (GitHub returns
`403 Resource not accessible by integration` for both `fork_repository`
and `create_repository` — App tokens cannot do either). The owner must
create the fork by hand:

1. Open `https://github.com/diwenne/openreply` → **Fork** → owner
   `brwarashidpour` (private or public — private is fine).
2. Result: `https://github.com/brwarashidpour/openreply`.
3. Then push the prepared local branch (see §7): the local clone already
   has branch `brva-phase1` with the rate-cap edit + these docs.

## 1. Generate secrets (commands only — run locally, paste values into hosts)

```bash
openssl rand -base64 32   # NEXTAUTH_SECRET  (min 16 chars)
openssl rand -base64 32   # CRON_SECRET
openssl rand -hex 32      # ENCRYPTION_KEY   (exactly 64 hex chars)
openssl rand -base64 32   # WEBHOOK_VERIFY_TOKEN (any random string; paste the same value into Meta)
```

- `ENCRYPTION_KEY` must be **identical** on the web app and the worker.
  Losing/changing it means every connected Instagram account must
  reconnect. **Back it up in the Secure Vault.**
- `ALLOWED_EMAILS` is pre-filled in `.env.example` with the owner's
  address so the public dashboard cannot be used by strangers to sign in.

## 2. Neon (PostgreSQL, free)

1. Create a Neon account + project (free tier, ~0.5 GB).
2. Copy the connection string from the Neon dashboard.
3. Use it as `DATABASE_URL` on **both** Vercel and the Botkeep worker.
   If Neon offers a pooled endpoint, use the pooled URL for runtime and
   the direct URL for migrations; otherwise the single URL works for both.

## 3. Redis Cloud (free)

1. Create a Redis Cloud account + Essentials database (free, 30 MB).
2. Copy the `redis://` / `rediss://` connection URL (it embeds the
   password — treat it as a secret).
3. Use it as `REDIS_URL` on **both** Vercel and the Botkeep worker.
   Must be native TCP Redis — BullMQ does not work over HTTP-only Redis.

## 4. Email for magic-link login (pick one)

- **Resend (free, 3k/mo):** create account + API key → `RESEND_API_KEY`;
  verify a sender domain → `EMAIL_FROM`.
- **Own SMTP (no Resend needed):** set `EMAIL_SERVER` to the SendPulse
  SMTP URL (`smtps://…:465`), URL-encoding special characters in the
  login/password (`@` → `%40`). `RESEND_API_KEY` is then not needed.

## 5. Vercel (web app)

1. Create a Vercel account → Add New Project → import the fork
   `brwarashidpour/openreply` (auto-detects Next.js).
2. Environment variables (exact names; values from §§1–4 + §6):
   `NEXTAUTH_URL` (= the Vercel domain, e.g.
   `https://<app>.vercel.app`), `NEXTAUTH_SECRET`, `CRON_SECRET`,
   `ENCRYPTION_KEY`, `DATABASE_URL`, `REDIS_URL`, `RESEND_API_KEY` (or
   `EMAIL_SERVER`), `EMAIL_FROM`, `ALLOWED_EMAILS`,
   `INSTAGRAM_APP_ID`, `INSTAGRAM_APP_SECRET`, `FACEBOOK_APP_SECRET`,
   `WEBHOOK_VERIFY_TOKEN`.
   - `DATABASE_URL` / `REDIS_URL` here must be the **public** URLs
     (Vercel runs outside any private network).
   - Optional: `DM_RATE_LIMIT_PER_HOUR` (default 75; only set to change it).
3. Deploy. The build runs `prisma generate && prisma migrate deploy &&
   next build`, so the database is migrated automatically.
4. Verify: `https://<app>.vercel.app/api/health` — database, Redis and
   queue should report healthy. (`worker.healthy` stays false until §6
   runs.)

## 6. Botkeep (worker)

1. In the Botkeep panel, create a **Node.js** server in a free slot,
   importing from GitHub: repo `brwarashidpour/openreply`, branch `main`
   (or `brva-phase1` until merged).
2. Set the start command to `npm run worker` (runs
   `tsx worker/dm-worker.ts`: BullMQ send queue + 5-min comment-polling
   reconciler + 30-s heartbeat). Make sure dependencies install and the
   Prisma client generates before start — if the panel has a separate
   build/install step, it must run `npm ci && npx prisma generate`.
   The worker needs the full source tree (`lib/`, `worker/`,
   `tsconfig.json` for the `@/*` alias, `node_modules`) — do **not**
   strip them.
3. Environment variables on the slot (same shared values as Vercel):
   `DATABASE_URL`, `REDIS_URL`, `ENCRYPTION_KEY`, `NEXTAUTH_URL`,
   `NEXTAUTH_SECRET`. (Meta secrets and `CRON_SECRET` are not needed by
   the worker. `DM_RATE_LIMIT_PER_HOUR` is optional; default 75.)
4. Verify in the panel console: `[DM Worker] Started`, then
   `/api/health` reports `worker.healthy: true`.

### 6a. Connectivity probe (the one open question — run BEFORE relying on it)

Botkeep's docs promise 24/7 operation with no scheduled idle sleep and
crash recovery, and the owner's Python auction-watcher already proves a
Botkeep slot stays alive and reaches the internet over HTTPS. What is
**not** documented is whether arbitrary outbound TCP (Neon Postgres on
5432/TLS, Redis Cloud on its TLS port) is allowed from a slot. Run this
from the slot console (or a one-off command) with the real hostnames
from the Neon/Redis dashboards before going live:

```bash
# Postgres (Neon) — expect a TLS handshake / server response, not a timeout
openssl s_client -connect <neon-host>:5432 -servername <neon-host> </dev/null
# Redis Cloud — expect a TLS handshake
openssl s_client -connect <redis-host>:<redis-port> -servername <redis-host> </dev/null
```

- If both connect: the $0 path works end to end.
- If either times out / is refused: the worker cannot reach that
  datastore from Botkeep → use the Railway paid fallback (worker +
  Postgres + Redis there, ~$5–10/month) and keep Vercel for the web app.

## 7. Push the prepared local branch (after §0)

The local clone already contains the Phase-1 changes on branch
`brva-phase1` (rate-cap edit + `.env.example` + these docs). Once the
fork exists:

```bash
git -C ~/workspace/skill-review/openreply remote set-url origin https://github.com/brwarashidpour/openreply.git
git -C ~/workspace/skill-review/openreply push -u origin brva-phase1
```

Then either merge `brva-phase1` into `main` on GitHub, or point Vercel /
Botkeep at the `brva-phase1` branch.

## 8. Meta app (owner, ~one afternoon)

Full steps: `docs/setup.md` ("The Meta app"). Summary:

1. `developers.facebook.com/apps` → create a **Business** app, use case
   **Manage messaging and content on Instagram** (not Marketing API,
   not Facebook Login).
2. Collect `INSTAGRAM_APP_ID`, `INSTAGRAM_APP_SECRET`
   (Instagram product → API setup with Instagram login),
   `FACEBOOK_APP_SECRET` (App settings → Basic), and choose
   `WEBHOOK_VERIFY_TOKEN`.
3. Add `@tir_vision` and `@brva.rashidpour` as **Instagram testers**
   (App roles → Roles) **and accept the invite inside Instagram**
   (profile → Settings → Apps and websites → Tester invites) for each
   account — both halves are required.
4. Register the OAuth redirect URI
   `https://<app>.vercel.app/api/instagram/callback` (no trailing slash).
5. Configure the webhook: callback
   `https://<app>.vercel.app/api/webhook` + the verify token; subscribe
   to the `comments` **and** `messages` fields.
6. Publish the app to **Live** (privacy / terms / data-deletion URLs
   are served by the app itself at `/privacy`, `/terms`,
   `/data-deletion`). No App Review is needed for the owner's own
   tester accounts.

## 9. Verify end to end (test account FIRST)

1. Connect the **test** Instagram account in the dashboard (Settings).
2. Create a campaign on one of its posts with keyword `TEST`.
3. From a **different** Instagram account, comment `TEST` on that post
   (OpenReply ignores an account's own comments).
4. Confirm the DM arrives and a `SENT` row appears in DM Logs, and
   `/api/health` shows `worker.healthy: true`.
5. **Do not touch the main pages yet. Do NOT enable “Take control of
   conversations” anywhere yet.**
6. Only then connect the main pages in order (see `AUTOMATIONS.md`):
   first `@brva.rashidpour` with automation B1 only, watch 48 h; then
   `@tir_vision` with A1. Enter the automations from `AUTOMATIONS.md`
   in the dashboard at that point — not before.

## Rollback

- Stop an automation: dashboard → campaign → `isActive = false`.
- Disconnect an account: dashboard → Settings → disconnect.
- Stop the worker: stop the Botkeep slot.
- Re-enable only after checking DM Logs / `/api/health` for the cause.

## Fork changes vs upstream (this branch)

- `lib/utils/rate-limiter.ts`: hourly DM cap 750 → **75** by default,
  overridable with `DM_RATE_LIMIT_PER_HOUR` (safe-parsed; bad values
  fall back to 75). Meta's documented cap is 750 — this fork runs at
  10% of it for headroom.
- `.env.example`: `ALLOWED_EMAILS` pre-filled with the owner's address;
  `DM_RATE_LIMIT_PER_HOUR` documented.
- Added: this file + `AUTOMATIONS.md` (dashboard-entry doc, no live
  automation created).
