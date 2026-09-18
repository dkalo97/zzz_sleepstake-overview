# ZzzStake

ZzzStake is a self-commitment app for sleep: you stake real money against your own sleep-score threshold, get it back if you hit it, and forfeit it (split between the company and a charity of your choice) if you miss. It's built on the same loss-aversion mechanic that makes stickK and Beeminder work for weight loss and habit change, applied to sleep instead — with a non-monetary social layer (leaderboards, streaks, friends) for engagement rather than pooled cash wagering between users. Currently a web app in active development, not yet processing real charges.

---

## Why is this repo private?

ZzzStake is pre-revenue but handles real user accounts, real sleep data, and is being built toward real financial transactions — the resolution engine that decides who wins or loses a stake, and the scoring formula that decides whether a night "counts," are the core IP and also the code most likely to cause real harm if it has a bug. I'd rather share progress and architecture openly than the exact implementation. I'm happy to do a live walkthrough or grant temporary read access — reach out via [LinkedIn](https://www.linkedin.com/in/danielkalo) or the contact on my resume.

---

## Technical Architecture

### Frontend — Next.js (App Router) + TypeScript

- **Next.js App Router, TypeScript, Tailwind v4, shadcn/ui** — server-rendered by default; auth state resolves in Server Components with zero client-side auth JS.
- **"Nocturnal precision"** — a dark-by-default design system chosen because the subject matter (sleep, night, stakes) calls for it, not as a lazy default.
- Pages: a dashboard, stake creation/detail/dispute flows, account settings, friends, and three leaderboard views (current streak, 30-day average, most improved), each in global and friends-only scope.

### Backend — Next.js API routes + Vercel Cron

- Route handlers under `app/api/` cover sleep-data ingestion, stake resolution, disputes, and account actions.
- A daily **Vercel Cron job** hits the resolution endpoint (secret-gated, not session-gated — it has no user to authenticate as) and runs every stake whose commitment window has closed.
- Ingestion is deliberately **client-agnostic**: `POST /api/sleep` accepts raw sleep-stage samples from any source (Apple Shortcuts, Health Auto Export, or eventually a native HealthKit bridge) rather than requiring a specific mobile client to exist first.

### Auth — hand-rolled (bcrypt + JWT), not a managed provider

Email/password and Google OAuth, both issuing short-lived access tokens and longer-lived refresh tokens as httpOnly cookies, with silent rotation. Chosen over a managed auth SaaS (evaluated and reversed) to match the rest of the portfolio's pattern and keep full control over the token lifecycle given what this app is eventually authorizing.

### Database — Neon (Postgres) + Drizzle ORM

Schema covers users, sleep-score records, stakes, a transaction ledger, and disputes, with separate branches for production and preview so schema changes can be verified in isolation before touching real data. A `(user_id, date)` uniqueness constraint on sleep records and a stored bedtime column exist specifically to make the resolution engine's inputs unambiguous.

### Sleep Score Engine — pluggable `ScoreProvider`

Apple doesn't expose its native Sleep Score through HealthKit, so ZzzStake computes its own from raw sleep-stage samples — a transparent, documented formula (duration, consistency, and interruption sub-scores) behind a `ScoreProvider` interface, so a future provider (a different device, a different formula version) can be swapped in without touching the resolution logic downstream. A night is canonically dated by its wake-up morning, and every date comparison anchors to the user's own timezone, not the server's — both were real bugs once, now enforced structurally.

### Resolution Engine — the part the "real money" rule is about

Pure win/forfeit decision logic, separated from its database orchestration, with the heaviest test coverage in the codebase. A multi-night stake only wins if *every* covered night clears the threshold. Nothing here has ever issued a real charge — resolutions currently write to an internal ledger recording what *would* be charged, so the logic can be validated against real behavior before any payment processor is wired in.

### Payments — architecture decided, not yet live

The planned model is card-on-file with charge-on-forfeit (the stickK/Beeminder pattern), not a pre-funded balance — chosen for compliance reasons, not preference. A fixed split of every forfeit is disclosed up front and routed partly to the company and partly to a user-chosen charity. No Stripe integration exists yet; this ships only once the legal and processor groundwork is actually confirmed, not just researched.

### Disputes

A full dispute/appeal flow lets a user contest a forfeited stake; an admin-gated review queue can reverse a decision. One dispute per stake, ever — reversing a forfeit doesn't reopen the door to dispute it again.

### Social — leaderboards & friends (opt-in)

Mutual-accept friendships, prefix search, and three leaderboard types, gated behind an explicit opt-in that controls both *appearing on* and *reading* the boards — you don't get to see others' data while withholding your own. Only usernames, sleep scores, and streaks are ever shared; never real names, email, or any dollar amount. Blocking severs a friendship and hides both users from each other's search and boards in both directions.

### Email — transactional notifications

Resolution outcomes and account flows (password reset, etc.) send real email via Resend, mirroring the same-day in-app banner copy.

---

## Project Structure

```
app/
  page.tsx                 # Dashboard
  stakes/new, stakes/[id]  # Stake creation, detail, dispute
  login, register,         # Auth flows (email/password + Google)
  onboarding, reset-password
  friends/                 # Friend requests, search
  leaderboard/             # Streak / average / most-improved boards
  account/                 # Profile, opt-ins, deletion
  disputes/                # Admin review queue
  api/
    sleep/                 # Client-agnostic sleep-data ingestion
    resolve/                # Cron-triggered resolution endpoint
    disputes/               # Dispute submission + admin resolution
components/
  auth/, friends/, leaderboard/, account/, ui/
lib/
  scoring/                 # ScoreProvider implementations, score formula
  resolution/               # Win/forfeit decision logic + orchestration
  disputes/                 # Dispute logic + orchestration
  stakes/                    # Stake lifecycle (creation, cancellation)
  charities/                 # Charity list + routing
  streaks/                   # Streak computation
  leaderboard/, friends/, blocks/  # Social layer
  auth/                       # Token issuance, verification, admin checks
  email/                      # Transactional email
  dates/                      # Timezone-correct calendar-day logic
  db/                         # Drizzle schema, migrations, seed data
```

---

## Status

Web app in active development, deployed continuously to `zzzstake.kalotech.dev`. No real payment processing yet — the resolution engine runs against real sleep data and a real ledger, but every forfeit today is a recorded intent, not a charge. Native mobile (Capacitor wrapping the same codebase, with a native HealthKit plugin) is planned once the core mechanic is validated against real personal data, not built from day one.

## Research foundation

The core mechanic isn't a guess: real stickK user data shows a 60-percentage-point lift in goal completion when a monetary stake is involved (79.1% vs. 53.1% success), consistent with loss aversion research showing losses are felt roughly 2.25x more intensely than equivalent gains. The effect is known to fade a few months after an incentive is removed, which is why the product is designed around recurring engagement (streaks, a social layer) rather than pitched as a one-time fix.
