# TSA LMS BUILD SPEC

**Project:** Transform Speaking Academy — self-hosted course platform + Bart integration
**Owner:** Alain Desaulniers
**Status:** Ready for execution
**Version:** 1.0
**Repo target:** existing TSA sales page app (Next.js), new route groups

---

## PART 0 — RULES FOR THE EXECUTING AGENT

Read this section fully before writing any code.

1. **Do not scaffold from a template.** This is an addition to an existing Next.js app. Read the existing app structure first and match its conventions.
2. **Do not invent design.** Design tokens are locked in Part 6. Use them exactly.
3. **No em dashes in any user-facing copy.** Anywhere. Including error messages.
4. **No AI-sounding language in user-facing copy.** No "unlock," "elevate," "journey," "dive in," "empower," "seamless," "unleash."
5. **Every database change is a migration file.** No manual dashboard edits to schema. `supabase/migrations/`.
6. **RLS is on for every table. No exceptions.** A table without a policy is a bug, not a default.
7. **Never trust the client for entitlement.** Every gated route re-checks server-side. A client-side check is a UX nicety, not a security boundary.
8. **Stop and ask** if a requirement here conflicts with something you find in the existing codebase. Do not resolve it yourself.
9. **Ship Phase 1 completely before starting Phase 2.** Do not build ahead.
10. **Write the tests listed in Part 9.** They are not optional.

---

## PART 1 — SCOPE

### In scope
- Course delivery: modules, lessons, video, rich text, progress tracking
- Auth: magic link primary, password optional
- Entitlements: perpetual course access, time-boxed then recurring Bart access
- Stripe checkout, webhooks, customer portal
- Bunny Stream signed playback
- Admin dashboard: users, progress, entitlements, Bart usage
- Bart embedded in lesson context, with logging and cost controls
- Transactional email

### Explicitly out of scope (v1)
- Community / forums / comments
- Live cohort scheduling
- Quizzes with grading
- Affiliate program
- Multi-course catalog (build for one course, structure for many)
- Mobile native apps (PWA only)

---

## PART 2 — ARCHITECTURE

```
┌─────────────────────────────────────────┐
│  Next.js App (Vercel)                   │
│  thetribe.rocks                         │
│                                         │
│  /              → sales page (existing)  │
│  /login         → auth                   │
│  /learn         → course shell (gated)   │
│  /learn/[slug]  → lesson view (gated)    │
│  /bart          → coach (gated, sep.)    │
│  /account       → billing, profile       │
│  /admin         → owner only             │
│  /legal/*       → terms, privacy, refund │
└────────┬──────────────────┬──────────────┘
         │                  │
    ┌────▼─────┐      ┌─────▼──────┐
    │ Supabase │      │  Stripe    │
    │ Postgres │      │  Checkout  │
    │ Auth     │      │  Portal    │
    │ RLS      │      │  Webhooks  │
    └──────────┘      └────────────┘
         │
    ┌────▼──────┐   ┌──────────┐   ┌─────────┐
    │  Bunny    │   │ Anthropic│   │ Resend  │
    │  Stream   │   │   API    │   │  Email  │
    └───────────┘   └──────────┘   └─────────┘
```

**Key principle:** Auth answers "who are you." Entitlements answer "what may you have." They are separate systems and must never be conflated.

---

## PART 3 — DATA MODEL

### Migration 001: core content

```sql
-- COURSES
create table courses (
  id uuid primary key default gen_random_uuid(),
  slug text unique not null,
  title text not null,
  description text,
  created_at timestamptz default now()
);

-- MODULES
create table modules (
  id uuid primary key default gen_random_uuid(),
  course_id uuid not null references courses(id) on delete cascade,
  slug text not null,
  title text not null,
  description text,
  position int not null,
  created_at timestamptz default now(),
  unique (course_id, slug),
  unique (course_id, position) deferrable initially deferred
);

-- LESSONS
create table lessons (
  id uuid primary key default gen_random_uuid(),
  module_id uuid not null references modules(id) on delete cascade,
  slug text not null,
  title text not null,
  position int not null,
  bunny_video_id text,
  duration_seconds int,
  body_mdx text,                 -- authored content
  bart_context text,             -- lesson-specific priming for Bart
  is_published boolean default false,
  created_at timestamptz default now(),
  unique (module_id, slug),
  unique (module_id, position) deferrable initially deferred
);

create index lessons_module_idx on lessons(module_id, position);
```

### Migration 002: users and entitlements

```sql
-- PROFILES (extends auth.users)
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  email text not null,
  full_name text,
  stripe_customer_id text unique,
  is_admin boolean default false,
  bart_logging_consent boolean default false,
  bart_research_consent boolean default false,
  created_at timestamptz default now()
);

-- ENTITLEMENTS  (the paywall)
create type product_key as enum ('tsa_course', 'bart');
create type entitlement_status as enum ('active', 'past_due', 'expired', 'revoked');

create table entitlements (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references profiles(id) on delete cascade,
  product product_key not null,
  status entitlement_status not null default 'active',
  expires_at timestamptz,              -- NULL = perpetual
  stripe_subscription_id text,
  granted_at timestamptz default now(),
  granted_by uuid references profiles(id),  -- null = automated
  notes text,
  unique (user_id, product)
);

create index entitlements_lookup on entitlements(user_id, product, status);
```

**Entitlement semantics — memorize this:**

| product | expires_at | stripe_subscription_id | meaning |
|---|---|---|---|
| `tsa_course` | `NULL` | `NULL` | perpetual, granted at purchase, never expires |
| `bart` | date | sub id | live until `expires_at`; Stripe renews it |

An entitlement is **live** if and only if:
```sql
status = 'active' AND (expires_at IS NULL OR expires_at > now())
```
This is the only definition. Put it in one function and call it everywhere.

```sql
create or replace function has_entitlement(p_user uuid, p_product product_key)
returns boolean language sql stable security definer as $$
  select exists (
    select 1 from entitlements
    where user_id = p_user
      and product = p_product
      and status = 'active'
      and (expires_at is null or expires_at > now())
  );
$$;
```

### Migration 003: progress

```sql
create table lesson_progress (
  user_id uuid not null references profiles(id) on delete cascade,
  lesson_id uuid not null references lessons(id) on delete cascade,
  started_at timestamptz default now(),
  completed_at timestamptz,
  last_position_seconds int default 0,
  updated_at timestamptz default now(),
  primary key (user_id, lesson_id)
);

create index progress_user_idx on lesson_progress(user_id);
create index progress_lesson_idx on lesson_progress(lesson_id) where completed_at is not null;
```

### Migration 004: Bart

```sql
create table bart_conversations (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references profiles(id) on delete cascade,
  lesson_id uuid references lessons(id) on delete set null,  -- null = standalone
  title text,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

create table bart_messages (
  id uuid primary key default gen_random_uuid(),
  conversation_id uuid not null references bart_conversations(id) on delete cascade,
  role text not null check (role in ('user','assistant')),
  content text not null,
  input_tokens int,
  output_tokens int,
  model text,
  latency_ms int,
  rating smallint check (rating in (-1, 1)),   -- thumbs down / up
  rating_note text,
  created_at timestamptz default now()
);

create index bart_msg_conv_idx on bart_messages(conversation_id, created_at);
create index bart_msg_rated_idx on bart_messages(rating) where rating is not null;

-- USAGE METER (cost control)
create table bart_usage (
  user_id uuid not null references profiles(id) on delete cascade,
  period_start date not null,          -- first of month
  message_count int default 0,
  input_tokens bigint default 0,
  output_tokens bigint default 0,
  estimated_cost_usd numeric(10,4) default 0,
  is_blocked boolean default false,    -- manual kill switch
  primary key (user_id, period_start)
);
```

### Migration 005: RLS

```sql
alter table profiles enable row level security;
alter table courses enable row level security;
alter table modules enable row level security;
alter table lessons enable row level security;
alter table entitlements enable row level security;
alter table lesson_progress enable row level security;
alter table bart_conversations enable row level security;
alter table bart_messages enable row level security;
alter table bart_usage enable row level security;

create or replace function is_admin()
returns boolean language sql stable security definer as $$
  select coalesce((select is_admin from profiles where id = auth.uid()), false);
$$;

-- PROFILES
create policy "own profile read"   on profiles for select using (id = auth.uid() or is_admin());
create policy "own profile update" on profiles for update using (id = auth.uid());

-- CONTENT: readable only with live course entitlement
create policy "course read"  on courses for select
  using (has_entitlement(auth.uid(), 'tsa_course') or is_admin());
create policy "module read"  on modules for select
  using (has_entitlement(auth.uid(), 'tsa_course') or is_admin());
create policy "lesson read"  on lessons for select
  using ((is_published and has_entitlement(auth.uid(), 'tsa_course')) or is_admin());

-- ENTITLEMENTS: read own, never write from client
create policy "own entitlements" on entitlements for select using (user_id = auth.uid() or is_admin());
-- no insert/update/delete policy = service role only. This is intentional.

-- PROGRESS: own only
create policy "own progress read"   on lesson_progress for select using (user_id = auth.uid() or is_admin());
create policy "own progress write"  on lesson_progress for insert with check (user_id = auth.uid());
create policy "own progress update" on lesson_progress for update using (user_id = auth.uid());

-- BART
create policy "own conversations" on bart_conversations for all
  using (user_id = auth.uid() or is_admin()) with check (user_id = auth.uid());
create policy "own messages" on bart_messages for select
  using (exists (select 1 from bart_conversations c
                 where c.id = conversation_id and (c.user_id = auth.uid() or is_admin())));
create policy "rate own messages" on bart_messages for update
  using (exists (select 1 from bart_conversations c
                 where c.id = conversation_id and c.user_id = auth.uid()));
create policy "own usage read" on bart_usage for select using (user_id = auth.uid() or is_admin());
```

**Note on `entitlements`:** there is deliberately no client write policy. All writes go through the service role in webhook handlers or admin server actions. If a client can write an entitlement row, the paywall does not exist.

---

## PART 4 — AUTH

**Supabase Auth. Enable both:**
- Magic link (primary, promoted in UI)
- Email + password (secondary, offered as "prefer a password?")

**Flow on purchase:**
1. Stripe webhook `checkout.session.completed` fires
2. Handler creates `auth.users` row via admin API if email is new
3. Handler creates `profiles` row, stores `stripe_customer_id`
4. Handler creates two `entitlements` rows (see Part 5)
5. Handler sends welcome email with magic link

**Never** rely on the user creating an account themselves. The purchase creates the account. Reduce every step between money and access.

**Admin bootstrap:** set `is_admin = true` manually via SQL for Alain's account. One row. Do not build an admin invite flow in v1.

---

## PART 5 — STRIPE

### Products to create in Stripe

| Stripe object | Price | Type |
|---|---|---|
| `TSA Course` | $1,997 | one-time |
| `TSA Course (3-pay)` | $797 × 3 | one-time, 3 installments |
| `Bart Coaching` | $TBD/mo | recurring, monthly |

### Checkout session shape

Single session, two line items:

```ts
const session = await stripe.checkout.sessions.create({
  mode: 'subscription',
  line_items: [
    { price: TSA_COURSE_PRICE_ID, quantity: 1 },   // one-time
    { price: BART_MONTHLY_PRICE_ID, quantity: 1 }, // recurring
  ],
  subscription_data: {
    trial_period_days: 180,
    trial_settings: {
      end_behavior: { missing_payment_method: 'cancel' }
    },
    metadata: { product: 'bart' }
  },
  customer_creation: 'always',
  metadata: { grants: 'tsa_course,bart' },
  success_url: `${SITE}/welcome?session_id={CHECKOUT_SESSION_ID}`,
  cancel_url: `${SITE}/#pricing`,
});
```

**Why this works:** card is captured at checkout, Bart runs a 180-day trial, Stripe charges automatically on day 181. No cron job. No expiry logic. Stripe is the scheduler.

Enable in Stripe dashboard:
- Trial ending reminder email: **15 days before** (day 165)
- Smart Retries for failed payments
- Customer Portal with cancel + card update enabled

### Webhook handlers (`/api/stripe/webhook`)

| Event | Action |
|---|---|
| `checkout.session.completed` | Create user + profile. Insert `tsa_course` entitlement (`expires_at: null`). Insert `bart` entitlement (`expires_at: now + 180d`, store sub id). Send welcome email. |
| `customer.subscription.updated` | Sync `bart` entitlement `expires_at` to `current_period_end`. Map Stripe status → entitlement status. |
| `customer.subscription.deleted` | Set `bart` entitlement `status = 'expired'`. **Do not touch `tsa_course`.** |
| `invoice.payment_failed` | Set `bart` status = `past_due`. Leave access live during Stripe's retry window. |
| `invoice.payment_succeeded` | Set `bart` status = `active`, extend `expires_at`. |
| `charge.refunded` | If course refunded: `tsa_course` → `revoked`, `bart` → `revoked`, cancel sub. |
| `charge.dispute.created` | Both → `revoked`. Alert Alain by email. |

**Hard requirements:**
- Verify signature with `stripe.webhooks.constructEvent`. Reject unsigned.
- Idempotency: store processed `event.id` in a `stripe_events` table, exit early on repeat. Stripe retries. You will get duplicates.
- Handler must respond 200 in under 5s. Do the work, then respond. If it gets slow, queue it.

### Customer Portal

```ts
const portal = await stripe.billingPortal.sessions.create({
  customer: profile.stripe_customer_id,
  return_url: `${SITE}/account`,
});
```
Link from `/account`. This is ~20 lines and removes most billing support forever. Build it in Phase 2, not later.

---

## PART 6 — DESIGN TOKENS (LOCKED)

```css
--ink:        #1A1A1A;   /* near-black, primary text + bg for dark surfaces */
--signal:     #F26522;   /* burnt orange, single accent, use sparingly */
--paper:      #FAF8F5;   /* warm off-white, primary bg */
--ink-60:     rgba(26,26,26,0.6);
--ink-12:     rgba(26,26,26,0.12);
```

**Type:**
- Display: Bricolage Grotesque / Archivo Expanded
- Body: system stack or existing app's body face. Match the sales page.

**Rules:**
- `--signal` is for one thing per screen. Progress fill, or the primary button, not both.
- Generous line height on lesson body. 1.7 minimum. People read these.
- Lesson content column: max 68ch. Do not let it run wide.
- No shadows-as-decoration. Borders and space.
- Short paragraphs in all copy.

---

## PART 7 — BUNNY STREAM

Never expose a raw Bunny URL. Server-side signed token per playback.

`/api/video/[lessonId]/token` (server route):
1. Auth check: session exists
2. Entitlement check: `has_entitlement(user, 'tsa_course')`
3. Fetch `bunny_video_id` for lesson
4. Generate signed token, expiry `now + 4 hours`
5. Return `{ url, expires }`

```ts
// Bunny token auth
const expires = Math.floor(Date.now() / 1000) + 14400;
const hash = crypto
  .createHash('sha256')
  .update(BUNNY_TOKEN_KEY + videoId + expires)
  .digest('hex');
```

Enable in Bunny dashboard: Token Authentication ON, Referrer whitelist to `thetribe.rocks`, direct URL access OFF.

Also turn on Bunny's automatic captions. Accessibility, and people watch on mute.

---

## PART 8 — BART INTEGRATION

### Route: `/api/bart/chat`

**Order of operations. Do not reorder:**

1. Auth check → 401
2. `has_entitlement(user, 'bart')` → 402 with re-subscribe link
3. `bart_usage.is_blocked` → 403
4. Monthly message cap check → 429 with reset date
5. Input length check (`MAX_INPUT_CHARS`) → 400
6. Load system prompt + lesson `bart_context` if `lesson_id` present
7. Load conversation history (**last 20 messages only** — hard cap, this is the cost lever)
8. Call Anthropic, stream response
9. On completion: write both messages, increment `bart_usage`, log tokens/latency/cost

### Cost controls (NOT OPTIONAL — build in Phase 1 of Bart work)

```ts
const LIMITS = {
  MAX_MESSAGES_PER_MONTH: 300,      // generous; median user ~40
  MAX_INPUT_CHARS: 8000,            // per message
  MAX_HISTORY_MESSAGES: 20,         // context window cap
  MAX_OUTPUT_TOKENS: 1500,
  SOFT_ALERT_USD: 15,               // email Alain when a user crosses this
  HARD_BLOCK_USD: 40,               // auto-set is_blocked, email Alain
};
```

**Rationale:** an uncapped LLM endpoint behind a $47/mo subscription is an open credit line. One user with a script costs more than a year of their subscription. The cap is a business requirement, not a technical one.

Show the user their usage in `/account`: "You've used 38 of 300 messages this month." Transparency prevents the support email.

### Consent

At first Bart use, a one-time modal. Plain language, no legalese:

> **Before we start**
>
> I save your conversations with Bart so you can come back to them, and so I can make Bart better.
>
> [ ] Save my conversations (required for Bart to remember context)
> [ ] Let Alain use anonymized excerpts to improve Bart and for research
>
> You can delete your history any time from your account page.

Store to `profiles.bart_logging_consent` and `bart_research_consent`. Second box is optional and unchecked by default. Do not use unconsented data for anything. Ever.

### Feedback

Thumbs up/down on every assistant message → `bart_messages.rating`. On thumbs down, optional one-line "what went wrong?" → `rating_note`.

**This is the point of the whole logging system.** A pile of transcripts is noise. A list of messages Alain's students rated bad is an eval set, and an eval set is how Bart actually gets better. Admin needs a view: all `rating = -1` messages, newest first, with full context.

---

## PART 9 — ROUTES

```
app/
  (marketing)/
    page.tsx                    existing sales page
    legal/terms/page.tsx
    legal/privacy/page.tsx
    legal/refunds/page.tsx
  (auth)/
    login/page.tsx
    welcome/page.tsx            post-purchase landing
  (app)/
    layout.tsx                  ← auth gate
    learn/
      page.tsx                  course home, resume CTA, progress
      [module]/[lesson]/page.tsx
    bart/
      page.tsx                  standalone coach
    account/
      page.tsx                  profile, usage, billing portal link, delete data
  (admin)/
    admin/
      layout.tsx                ← is_admin gate
      page.tsx                  overview stats
      users/page.tsx
      users/[id]/page.tsx       per-user detail
      content/page.tsx          lesson drop-off funnel
      bart/page.tsx             usage + thumbs-down review
  api/
    stripe/webhook/route.ts
    stripe/portal/route.ts
    video/[lessonId]/token/route.ts
    bart/chat/route.ts
    progress/route.ts
```

**Gating:** `(app)/layout.tsx` checks session only. Each page checks its own entitlement server-side. `/learn/*` needs `tsa_course`. `/bart` needs `bart`. `/account` needs neither beyond auth.

---

## PART 10 — ADMIN DASHBOARD

Four screens. Resist building more.

**1. Overview**
- Active students (live `tsa_course`)
- Active Bart subs / in trial / past due
- MRR (Bart) and course revenue this month
- Median course completion %
- Bart spend this month vs Bart MRR (the margin number)

**2. Users**
Table: name, email, joined, course progress %, Bart status, Bart messages this month, Bart cost this month. Sortable. Searchable.

Row click → detail: full lesson-by-lesson progress, entitlement history, Bart conversation list, and buttons: `Grant course`, `Extend Bart`, `Revoke`, `Block Bart`, `Open in Stripe`.

**3. Content funnel**
Per lesson, in course order: started / completed / completion %. Render as a bar list.

> The lesson where the bar falls off a cliff is the lesson that is broken. This screen is worth more than the rest of the admin panel combined.

**4. Bart review**
- Usage leaderboard (spot the cost outliers before the bill)
- All `rating = -1` messages, newest first, full conversation context, exportable to JSON

---

## PART 11 — EMAIL (Resend)

| Trigger | Email |
|---|---|
| Purchase | Welcome + magic link. One button. Nothing else. |
| Magic link request | Sign-in link |
| Day 165 | Bart trial ending (Stripe sends this, configure it) |
| Payment failed | Update your card (Stripe sends) |
| 7 days inactive, course incomplete | One nudge. One. |
| Course complete | Congratulations + certificate |

**Do not build a drip sequence.** You have a promo email system already. This is transactional only.

The inactivity nudge should name the specific next lesson, not say "come back." Binary and specific: *"You're one lesson from finishing Module 2. It's 11 minutes."*

---

## PART 12 — BUILD ORDER

### Phase 1 — The spine
- [ ] Migrations 001-005, all RLS policies
- [ ] Supabase Auth, magic link + password
- [ ] `(app)` layout auth gate
- [ ] `/learn` shell: sidebar, lesson list, progress bar
- [ ] Lesson view: Bunny player, MDX body, mark complete
- [ ] `/api/video/[lessonId]/token`
- [ ] `/api/progress`
- [ ] Seed one real module of real content
- **Gate: a manually-inserted entitlement row gives real access to real video. Nothing else counts.**

### Phase 2 — The money
- [ ] Stripe products + prices
- [ ] Checkout session route
- [ ] Webhook handler, all events in Part 5, with idempotency
- [ ] `stripe_events` table
- [ ] Customer Portal route + `/account`
- [ ] Welcome email
- [ ] Legal pages (terms, privacy, refunds) live before first real sale
- **Gate: test-mode purchase creates user, grants both entitlements, sends email, user lands in course. Refund revokes.**

### Phase 3 — Your side
- [ ] Admin gate + four screens (Part 10)
- [ ] MDX lesson authoring workflow, documented in repo README
- **Gate: Alain can see who's stuck and manually fix any access problem without SQL.**

### Phase 4 — Bart (the reason)
- [ ] `bart_usage` + all limits from Part 8 **first, before the chat route**
- [ ] Consent modal
- [ ] `/api/bart/chat` with entitlement + cap checks
- [ ] `/bart` standalone
- [ ] Bart panel inside lesson view, primed with `bart_context`
- [ ] Thumbs up/down
- [ ] Admin Bart review screen
- **Gate: entitlement expiry actually blocks Bart. Cap actually caps. Thumbs-down actually lands in admin.**

### Phase 5 — Polish
- [ ] PostHog: funnel + session replay
- [ ] Certificate on completion
- [ ] PWA manifest (mirror Command Council Dashboard setup)
- [ ] Inactivity nudge
- [ ] `/account` data delete

---

## PART 13 — TESTS (required)

Do not skip these. They cover the places where money and access meet.

1. Expired `bart` entitlement → `/bart` returns 402, `/learn` still returns 200
2. Revoked `tsa_course` → `/learn` returns 402
3. Duplicate webhook `event.id` → no duplicate entitlement rows
4. `charge.refunded` → both entitlements revoked, sub cancelled
5. `customer.subscription.deleted` → `bart` expired, `tsa_course` untouched
6. Video token route without entitlement → 403, no Bunny URL in response body
7. User A cannot read User B's `lesson_progress` (RLS, test with anon key)
8. User A cannot read User B's `bart_messages` (RLS, test with anon key)
9. Client cannot insert an `entitlements` row with anon key
10. Bart cap at limit → 429, no Anthropic call made
11. Bart history never exceeds 20 messages regardless of conversation length

Tests 7, 8, 9 must run against the **anon key**, not the service role. Testing RLS with the service role tests nothing.

---

## PART 14 — ENV

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=        # server only. never NEXT_PUBLIC_.
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_COURSE_PRICE_ID=
STRIPE_COURSE_3PAY_PRICE_ID=
STRIPE_BART_PRICE_ID=
BUNNY_LIBRARY_ID=
BUNNY_TOKEN_KEY=
ANTHROPIC_API_KEY=
RESEND_API_KEY=
NEXT_PUBLIC_SITE_URL=
NEXT_PUBLIC_POSTHOG_KEY=
```

---

## PART 15 — COST MODEL

**Fixed:** ~$50-60/mo (Vercel Pro $20, Supabase Pro $25, Bunny ~$5-15, Resend $0, domain ~$1)

**Variable:**
- Stripe: 2.9% + $0.30 (~$58 on a $1,997 sale)
- Bart API: ~$2-5/active user/mo typical, $10-15 heavy

**The number to watch:** Bart spend vs Bart MRR, on the admin overview. If that ratio ever goes above ~25%, the caps are wrong or the price is wrong.

---

## OPEN DECISIONS (Alain)

1. **Bart monthly price.** Nothing downstream is blocked, but Stripe product creation needs it in Phase 2.
2. **Refund window.** 14 days? 30? Needs to be on the legal page before first sale.
3. **3-pay + Bart trial interaction.** If someone is on 3-pay, does Bart's 180 days start at first payment or last? Recommend: first. Simpler, more generous, better story.
4. **Does course access survive a Bart cancel?** Spec says yes. Confirm.
5. **Free tier of Bart for non-students?** Recommend no in v1. It's the thing that makes TSA worth buying.
