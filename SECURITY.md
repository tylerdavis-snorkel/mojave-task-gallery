# Security — Mojave Task Idea Gallery

What protects the data, where the enforcement actually lives, and what it
deliberately does not cover.

The gallery is a static HTML page talking to Supabase from the browser. There
is no server of our own, so **every meaningful control lives in the database**.
Anything enforced only in JavaScript is a convenience, not a defence — a user
can edit the page in devtools.

---

## Authentication

- **Google OAuth via Supabase Auth.** We never see, store, or transmit a
  password. Sessions are JWTs managed by the Supabase client.
- **No anonymous access.** The app queries nothing without a session; it shows
  the sign-in gate instead. Even if it did query, `anon` holds no table grants,
  so the request would be refused.
- **Google client secret** exists only inside Supabase's provider config. It is
  not in the repo, the HTML, or any file we produced.
- **Redirect allowlist.** Supabase only returns tokens to URLs listed in Auth →
  URL Configuration, so a lookalike site can't complete the flow.

---

## Authorisation

Admin rights come from the `mojave_admins` table, checked two ways:

| Layer | Mechanism | What it does |
|---|---|---|
| Database | `is_mojave_admin()` — `security definer`, matches `auth.jwt()->>'email'` | **Real enforcement.** Used inside RLS policies. |
| Browser | `ADMIN_EMAILS` loaded from the same table | Shows/hides admin tabs. Cosmetic only. |

Editing the JS array in devtools reveals the admin tabs but grants nothing —
every write behind them is refused by RLS.

---

## Row Level Security

RLS is enabled on all five tables.

| Table | Read | Write |
|---|---|---|
| `mojave_ideas` | any signed-in user | admins only |
| `mojave_claims` | any signed-in user | insert/update your own rows, or admin |
| `mojave_claim_events` | **admins only** | append your own events, or admin |
| `gallery_visits` | **nobody** — no select policy exists | insert your own row only |
| `mojave_admins` | any signed-in user | **nobody** — no write policy exists |

Two consequences worth stating plainly:

- A contributor **cannot read the claim log**, so they can't see who else claimed
  what, or with which intent.
- Visit logs are **write-only from the browser**. They are readable only through
  the Supabase dashboard or a direct database connection.

Realtime subscriptions run through the same policies, so live updates never
push a row a user couldn't already query.

---

## Grants

RLS decides *which rows*; grants decide *whether the table is reachable at all*.
The `authenticated` role holds only what the app needs — `select` on ideas and
admins, `select/insert/update` on claims, `select/insert` on events, `insert` on
visits. `anon` holds nothing.

`TRUNCATE` was revoked from `authenticated`. RLS does not apply to TRUNCATE, so
that privilege would have allowed emptying a table outright had it been
reachable. (It isn't exposed through PostgREST, but it had no business being
granted.)

---

## Integrity constraints

Enforced by the database, so they hold regardless of what the client does:

- `display_id` **unique** — no duplicate idea IDs.
- `lower(snorkel_submission_id)` **unique**, partial — a submission ID can be
  used once across the whole project, case-insensitively. This is the one the
  browser genuinely cannot enforce: under RLS a contributor can't see other
  people's claims, so only the database can catch a collision.
- **One live claim per idea** and **one live claim per person** — partial unique
  indexes covering active statuses only, so released claims never block a
  re-claim.
- `task_id` foreign key with `on delete cascade`.

---

## Client-side hardening

- **XSS.** All database content passes through `escapeHtml()` / `escapeAttr()`
  before rendering — 74 call sites. The handful of unescaped interpolations are
  in `window.confirm()` strings and clipboard text, which are plain-text
  contexts, not HTML.
- **Submission IDs** are validated against a UUID v4 pattern before submission.
- **No browser storage of app data.** No `localStorage`, no `sessionStorage`.
  Only the Supabase client's own session handling.
- **No `service_role` key anywhere in the client.** That key bypasses RLS
  entirely and lives only in Supabase.

---

## Data minimisation

- Visit logging records **email and timestamp only**. No duration, no
  heartbeat, no IP, no user agent. This was a deliberate narrowing.
- The claim log records email, idea, event, intent. No free text about people.
- No analytics, trackers, or third-party scripts beyond the Supabase client
  library from jsDelivr.

---

## Known limitations

Stated because pretending otherwise is worse than the limitation.

**Blurred tiles are a visual control, not an access control.** An idea claimed
by someone else is blurred with CSS, but the text remains in the DOM and is
readable via devtools. All signed-in users can read all ideas by policy —
hiding claimed ones properly requires filtering server-side, e.g. a view that
omits the body of ideas you don't hold.

**Anyone with a Google account can sign in.** The OAuth consent screen is
External so contributors can use their own accounts. There is no contributor
allowlist, so any Google user who finds the URL can sign in, browse ideas, and
claim one. If that matters, the fix is an allowlist table checked in RLS, the
same pattern as `mojave_admins`.

**Admin actions are attributed but not audited.** Blocks, approvals and
unblocks are written to the claim log, but editing an idea overwrites it with
no history of the previous version.

**The anon key is public.** By design — it identifies the project, it doesn't
authorise anything. All access is decided by the user's JWT and the policies
above. It appearing in a public repo is expected and safe.

**Third-party CDN.** The Supabase client loads from jsDelivr. A compromise
there would be a compromise of the page. Pinning a Subresource Integrity hash,
or vendoring the library into the repo, would close that.

---

## If credentials are exposed

| Leaked | Impact | Action |
|---|---|---|
| anon key | none | nothing |
| Google client secret | someone could impersonate the sign-in app | rotate in Google Cloud, update Supabase |
| database password | full read/write, RLS bypassed | Settings → Database → Reset password |
| `service_role` key | full read/write, RLS bypassed | rotate immediately in Settings → API |

The database password and `service_role` key are the two that matter. Neither
appears in the repo, the HTML, or any generated file.
