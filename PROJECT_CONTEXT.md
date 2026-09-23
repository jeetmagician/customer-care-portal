# Namonamaha — Customer Care Portal: project context

**Read this first, before touching anything.** This file exists so a fresh Claude
session — with no memory of the conversations that built this app — can understand
what already exists here before you start describing new work, especially work on
the **separate** `admin-portal` project (see [§12](#12-relationship-to-the-separate-admin-portal-project)
for why that distinction matters).

If you're the user: paste this whole file into a new conversation (in this repo or
in `admin-portal`) to give Claude the context it needs.

---

## TL;DR — what's in this portal, end to end

1. **One login page** (`index.html`, `localhost:8000`). Admin (`admin` / `Admin@123`)
   and temple customer-care staff sign in on the same form.
2. **Admin** lands on a single page, **Temple customer care**: every temple (8) with
   its CC login(s). IDs/passwords are masked (`pr*****` / `Ka*****`) until admin
   clicks *Show / change*; admin can edit ID, password, temple, and active flag.
   There is **no "add login"** here (removed on purpose). Clicking a temple opens a
   **sign-in pop-up** that accepts only that temple's own credentials.
3. **Temple customer care** (per temple, distinct password each, e.g. Kamakhya
   `priya.care` / `Kamakhya@101`): a CRM scoped to that temple only — Work queue,
   All history (table + CSV export), SLA & escalations, Cancellations, Message log,
   auto-assign agent, edit booking, confirm call, send balance link / OTP.
   Isolation covers bookings, the Activity feed, and the (hidden) role tabs.
4. **Devotee privacy in CC**: names `Sur**** Cho****`, phones `+91 9000*****`,
   Internet-call only (no direct Call / WhatsApp to devotees). Agent/pandit contact stays real.
5. Other roles (Devotee, Agent, Pandit mobile-style apps) exist in the file but are
   only reachable by a signed-in admin via the top role tabs.
6. All data is in-memory + `localStorage` (`namonamaha-care-demo-v11`). No backend.

## What to build in the separate `admin-portal` (suggested scope)

This repo already prototypes the admin side of temple customer care; the real
`admin-portal` should own it properly. Build there:
- **Real authentication** for admins (hashed passwords, sessions) — this repo's admin
  login is a plaintext demo check.
- **Temple customer care management**: list all temples with a **search box** (100s of
  temples) and collapsible logins; create / edit / deactivate a temple's CC login;
  admin-set passwords stored **hashed**, never displayed in full (mask by default, reset
  instead of reveal is safer); one credential set per temple, uniqueness enforced.
- **"Open temple CC"** as a real handoff: a signed, short-lived link/token from
  admin-portal to this customer-care app (never a bare URL param), plus the
  per-temple sign-in step this repo already does. Needs a shared backend.
- A **shared backend/database** (bookings, agents, pandits, CC users, temples) so both
  apps read the same data — today each has its own in-memory copy.
- Admin views this repo dropped from its sidebar and that belong in admin-portal:
  Overview, Bookings, Details of devotees (unmasked, with export), Temples, Puja
  catalogue, Payout rules, Agents & pandits (roster, phones, capacity, temple
  coverage), Blackout calendar, Payments & settlement, Roles & permissions, Settings.
  (Their reference implementations still live in `index.html`: `adDash`, `adBookings`,
  `adDevotees`, `adPeople`, `adPayments`, …)
- Keep the **rules from this portal**: temple isolation for CC, devotee masking in CC,
  admin sees real data.

## 1. What this is

**Namonamaha** is a temple-puja booking service. A devotee books a puja online or
by phone; an onsite agent meets them at the temple; a pandit performs the rite.
This repo, `customer-care-portal`, is a **single-file, self-contained prototype**
of the internal tooling around that flow — polished enough for a stakeholder
walkthrough, explicitly **not** production code (in-memory state, plaintext demo
passwords, simulated payment gateway, etc., all labelled as such in the UI itself).

It is **one file**: `index.html` (~350KB — inline `<style>`, inline `<script>`,
no build step, no framework, no dependencies). The only other files are three
images (`logo.jpg`, `logo-transparent.png`, `wallpaper.jpg`) referenced by
relative path.

## 2. Tech stack & how to run

Vanilla JS, vanilla CSS, no bundler. Served locally with:
```
python3 -m http.server 8000
```
then opened at `http://localhost:8000`. Deployed by pushing to
`https://github.com/jeetmagician/customer-care-portal` (`main` branch).

## 3. Architecture conventions

The `<script>` is organised top-to-bottom as:
```
CONFIG → DATA → STORE → UI → SCREENS → ROUTER
```
- **CONFIG/DATA**: constants and the seed arrays (`TEMPLES`, `PUJAS`, `AGENTS`,
  `PANDITS`, `CARE_USERS`, `ADMIN_USERS`, `BLACKOUTS`) — all top-level `let`
  globals, populated once by `seed()`.
- **STORE**: `DB` (bookings + events) plus every function that mutates a booking.
  Nothing outside this section writes to a booking directly — everything goes
  through `setField`/specific action functions (`assignAgent`, `payAdvance`,
  etc.) so the audit trail (`b.audit`) and the event feed (`DB.events`) never
  drift from the data.
- **UI**: shared render helpers (`kv`, `field`, `contactIconBtns`, `dashShell`,
  `trig`, …).
- **SCREENS**: one function per screen, keyed by role and sub-tab (e.g.
  `ccQueue`, `ccBooking`, `adDevotees`).
- **ROUTER**: a single global mutable state object `S` (current role, sub-tab,
  open modal, form drafts, …), one `render()` that does a **full-tree
  `innerHTML` replace** of `#app` on every state change, and one delegated
  `document.addEventListener('click', …)` that dispatches on a `data-act`
  attribute (plus separate `change`/`input` listeners for live form fields).

**State persistence**: everything in `S` is transient (reset on every page
load). What persists is the demo business data — `DB`, `TEMPLES`, `PUJAS`,
`AGENTS`, `PANDITS`, `CARE_USERS`, `ADMIN_USERS`, `BLACKOUTS` — serialized to
`localStorage` under a versioned key, currently `namonamaha-care-demo-v11`
(`STORAGE_KEY`). **Any change to the shape of one of those arrays must bump
this version number**, or a browser with old cached data will silently keep
replaying stale state forever (`seed()` always runs first, then
`restoreDemo()` unconditionally overwrites it with whatever was last saved).
This has caused real, confusing bugs before — treat the bump as a mandatory
part of the change, not an afterthought.

## 4. Roles & screens

Five roles, switched via top tab bar (`ROLE_TABS`): **Devotee** (`user`),
**Customer care** (`cc`), **Agent** (`agent`), **Pandit** (`pandit`). **Admin**
(`admin`) is a fifth role, not in the top tab bar — admins sign in through the
same login form as customer care staff (see §8).

The top role tabs are shown **only to a signed-in admin** (`S.adminUser`); temple
customer care staff never see them, so they can't reach other roles' screens.

## 5. Data model (the globals)

- **`TEMPLES`** (8) — id, name, city/state, live flag, opening hours, which
  service modes it supports.
- **`PUJAS`** (20: `P1`–`P12` in-person, `O1`–`O3` online, `S1`–`S5`
  subscription) — id, templeId, mode, price, lead time, payout split
  (`pujari`/`samagri`/`agent`/… as fixed amounts or percentages).
- **`AGENTS`** (20) — id, name, phone, which temples they cover, active flag,
  shift string.
- **`PANDITS`** (~83) — id, name, phone, home temple, `agentIds` (which
  agent(s) they work under), skills, daily capacity, leave flags.
- **`CARE_USERS`** — Customer Care staff logins (`id`, `name`, `loginId`,
  `password` — plaintext, explicitly flagged as demo-only). Managed **only**
  from Admin → Temple customer care; there is no self-service signup or password
  change. Everyone signs in through the same single login form.
- **`ADMIN_USERS`** — a separate, higher-privilege account set (currently one
  seeded account, `admin` / `Admin@123`). Admins sign in through the shared login form (§8).
- **`BLACKOUTS`** — temple-closure date ranges that block or warn on new
  bookings.
- **`DB.bookings`** — the actual booking records (see §6). **`DB.events`** —
  a simulated outbound-message log (Push/WhatsApp/Email/System/Payment),
  rendered in the global "Activity" drawer available from every screen.

A booking (`mkBooking()` factory) carries: identity (`id`, `clientId`,
`poojaBookingId`), the `devotee` sub-object (`name`, `gotra`, `phone`,
`email`, `city`, `dob`, `pob`, `wish`), which temple/puja/agent/pandit it's
assigned to, money fields (`amount`, `advanceAmt`, `balanceAmt`, paid flags),
status (see §6), OTP/arrival codes, `callLog`, `timeline`, `audit`,
`reminders`.

## 6. Booking lifecycle

`DRAFT → ADVANCE_PAID → CC_CONFIRMED → LINK_SENT → FULLY_PAID →
PANDIT_ASSIGNED → (running) → COMPLETED → SETTLED`, with `CANCELLED` and
`escalated` as side-states. Customer Care's job, in order: the confirmation
call → send the balance-payment link → (agent assigns a pandit, or CC
overrides after 24h) → send the completion OTP → (Accounts settles payouts
in Admin). Field-level permissions per role live in the `PERMS` table
(`can(role, path)` → `edit`/`read`/`lock`/`masked`/`hide`/`flow`), rendered
generically by `field()`.

## 7. Contact system (Call / Internet call / WhatsApp)

Every person's phone number gets a small icon row (`contactIconBtns` — real
`tel:` Call, a simulated "Internet call" masked-line button, and a `wa.me`
WhatsApp link), used consistently everywhere a name appears: Devotee, Agent,
Pandit alike — **except** for one deliberate carve-out described next.

## 8. Login flow and temple-wise customer care

**One login form** (`careLoginScreen`) is the entry point. `cc-login` checks
`ADMIN_USERS` first, then `CARE_USERS`:
- **Admin** (seed: `admin` / `Admin@123`) → lands directly on the Admin panel,
  which now has a **single tab, "Temple customer care"**. `adminScreen()` is
  login-gated (`S.adminUser`); the old footer link to admin was removed and the
  other admin tab functions (`adDash`, `adBookings`, `adPeople`, …) still exist
  in the file but are no longer reachable from the sidebar.
- **Temple staff** (`CARE_USERS[].templeId`) → land straight in their own
  temple's portal.

**Temple-wise**: each temple (T1–T8) has its own CC login(s), each with a
**different password** (seeded `Kamakhya@101`, `Bagala@102`, …). A CC session is
scoped to its temple via `ccTempleId()` / `scopeBk()` (queue, All history, SLAs,
cancellations, auto-assign).

**Admin opening a temple's portal**: Temple customer care lists every temple
with its logins (admin sees and can edit every ID, password and temple). Clicking
"Open <temple> customer care →" (`ad-open-cc`) shows a **sign-in pop-up**
(`templeLoginModal`, state `S.templeLogin`) that only accepts that temple's own
credentials (`ad-temple-login`). On success the temple's CC portal opens with
`S.fromAdmin=true`, showing a "← Back to admin" button (`cc-back-admin`) instead
of Log out. Admin has no bypass — by design, the temple's password gates it.

History: a Desk A/B split with a seat lock was built then removed by the user;
don't reintroduce it. Roster: 20 agents, ~83 pandits (`growRoster()`).

The separate `admin-portal` project can't do this itself (different origin, no
shared data); it would need a backend with real per-temple credentials/tokens.

## 9. Devotee PII masking (Customer Care only)

Driven by `isCareMasked()` (`S.role==='cc'`) and two functions:
- `maskName(name)` — keeps the first 3 letters of each word, then 4 literal
  asterisks (e.g. `"Suranjeet Choudhury"` → `"Sur**** Cho****"`). This is a
  **uniform rule Claude chose**, not a literal per-character spec from the
  business — the original example given wasn't internally consistent (3
  letters kept on one word, 2 on the other), so a consistent rule was picked
  instead. Revisit if the business wants an exact different rule.
- `maskPhoneDisplay(phone)` — keeps the country-code prefix + first 4 digits
  of the local number, then 5 asterisks (e.g. `"+91 90000 30041"` →
  `"+91 9000*****"`; a bare `"7002451825"` → `"7002*****"`, which matches the
  business's own example exactly).

`devDisplayName(b)`/`devDisplayPhone(b)` wrap these and are used **everywhere**
a devotee's identity is shown inside CC screens: list rows (`ccRow`), the
booking detail page (`ccBooking`), the temple/agent staffing views, the
auto-assign results panel, the AI-generated call note (`aiNote`), and even
the global cross-role "Activity" event drawer (`renderEvents`/
`maskEventText` substitutes the real name out of the stored event text
**at render time only, per-viewer** — the stored `DB.events` text itself
always keeps the real name, since Admin and other viewers must still see it
unmasked).

**Contact restriction**: inside CC, a devotee's contact row is
`devoteeContactCC(maskedName)` — **Internet call only**, no `tel:` Call
link, no WhatsApp link (removed deliberately, so a remote handler can never
walk away with — or misdial/message — the real number). Agent and pandit
contact are **untouched** everywhere, including inside CC screens — the
concern was specifically about the client's own number reaching a remote
handler, not staff-to-staff contact. CSV export from the Devotees directory
(`devoteesCsv`) also masks when `isCareMasked()` is true, so export can't be
used to route around the on-screen masking.

**PERMS**: `booking.devotee.name` / `booking.devotee.phone` are
`care:'masked'` (was `'lock'`) in the `PERMS` table — `admin` is always
`'edit'` regardless of this table (see `can()`), so Admin is unaffected.

## 10. Admin panel — current state

Only one sidebar tab is reachable: **Temple customer care** (`adCareTeam()`), see §8
and the TL;DR. Masked credentials with a per-person *Show / change* toggle
(`maskCred`, `S.revealCred`); no add-login form. The other admin screens' code
(`adDash`, `adBookings`, `adDevotees`, `adTemples`, `adPujas`, `adPayouts`,
`adPeople`, `adBlackouts`, `adPayments`, `adPerms`, `adSettings`) is still in the
file as reference but not wired into the sidebar.

Everything admin edits is read live by the customer care screens from the same
in-memory arrays — no sync step, same objects.

## 11. Known gaps / open items in *this* app

- The admin login is a client-side check against plaintext demo passwords
  (`ADMIN_USERS`, seed `admin` / `Admin@123`) — fine for a demo, not real
  security. Real auth needs a backend.
- The Admin sidebar is a single tab; other admin screens are unreachable (§10).
- Anyone reading the page source/localStorage can see the plaintext demo
  passwords — masking is a UI measure only.
- With 100s of temples the Temple customer care list will need a search box
  and collapsible logins.
- The header search bar (`.dash-search`, "Search bookings, devotees,
  pujas…") is permanently `disabled` — decorative only, never wired up.
- Pandit count is ~83 rather than a round 100 (§8) — cosmetic, not
  functional.

## 12. Relationship to the separate `admin-portal` project

**`admin-portal` is a completely different codebase** at
`/Users/suranjeet/admin-portal/`, built and maintained separately by the
user. It is *not* the "Admin panel" role described in §8 above, which
lives entirely inside this repo's `index.html`. Do not assume Claude working
in `admin-portal` has any access to this repo, its `DB`/`AGENTS`/`PANDITS`
arrays, or anything else described in this file — they're different
origins, different processes, and were explicitly kept unsynced by the
user's own choice ("I'll edit that admin portal separately later on"). If
work in `admin-portal` needs to reflect concepts from here (the masking rules, the data shapes), **read this file for the
concepts, then re-implement them against `admin-portal`'s own actual code**
— don't assume any file, function or variable name here exists over there
without checking.
