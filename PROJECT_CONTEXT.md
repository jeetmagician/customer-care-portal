# Namonamaha — Customer Care Portal: project context

**Read this first, before touching anything.** This file exists so a fresh Claude
session — with no memory of the conversations that built this app — can understand
what already exists here before you start describing new work, especially work on
the **separate** `admin-portal` project (see [§12](#12-relationship-to-the-separate-admin-portal-project)
for why that distinction matters).

If you're the user: paste this whole file into a new conversation (in this repo or
in `admin-portal`) to give Claude the context it needs.

---

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
`localStorage` under a versioned key, currently `namonamaha-care-demo-v8`
(`STORAGE_KEY`). **Any change to the shape of one of those arrays must bump
this version number**, or a browser with old cached data will silently keep
replaying stale state forever (`seed()` always runs first, then
`restoreDemo()` unconditionally overwrites it with whatever was last saved).
This has caused real, confusing bugs before — treat the bump as a mandatory
part of the change, not an afterthought.

One separate, **deliberately non-versioned** localStorage key exists:
`nm_care_seats` (see §8) — it's live session-coordination state, not demo
business data, so it isn't part of the save/restore blob.

## 4. Roles & screens

Five roles, switched via top tab bar (`ROLE_TABS`): **Devotee** (`user`),
**Customer care** (`cc`), **Agent** (`agent`), **Pandit** (`pandit`). **Admin**
(`admin`) is a fifth role that exists but is deliberately **not** in the top
tab bar — the only way in is a footer link on the Customer Care login screen
("Care team access"). See §11 for what Admin can do, and the important caveat
about it having no login gate yet.

The top role tabs themselves are **hidden entirely** until Customer Care is
signed in (`S.role!=='cc' || !!S.careUser` gates whether `#roles` renders
anything) — a deliberate choice so an unauthenticated visitor can't jump
straight to the Agent/Pandit/Devotee views from the CC login screen.

## 5. Data model (the globals)

- **`TEMPLES`** (8) — id, name, city/state, live flag, opening hours, which
  service modes it supports.
- **`PUJAS`** (20: `P1`–`P12` in-person, `O1`–`O3` online, `S1`–`S5`
  subscription) — id, templeId, mode, price, lead time, payout split
  (`pujari`/`samagri`/`agent`/… as fixed amounts or percentages).
- **`AGENTS`** (20) — id, name, phone, which temples they cover, active flag,
  shift string, and **`careTeam:'A'|'B'`** (see §8).
- **`PANDITS`** (~83) — id, name, phone, home temple, `agentIds` (which
  agent(s) they work under), skills, daily capacity, leave flags. A pandit's
  desk is **derived**, not stored: `panditTeam(d)` looks up
  `agent(d.agentIds[0]).careTeam`.
- **`CARE_USERS`** — Customer Care staff logins (`id`, `name`, `loginId`,
  `password` — plaintext, explicitly flagged as demo-only). Managed **only**
  from Admin → Care team access; there is no self-service signup or password
  change. The desk (A/B) is **not** part of a `CARE_USER` record — it's chosen
  at sign-in time (§8).
- **`ADMIN_USERS`** — a separate, higher-privilege account set (currently one
  seeded account, `admin` / `Admin@123`). See §11 for the important gap here.
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
`reminders`, and — new — **`careTeam:'A'|'B'`**, assigned round-robin at
creation (`bookingNo%2`) so clients are split evenly between the two desks
the instant they book, independent of when/whether an agent gets assigned.

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

## 8. The two customer-care-desk system

This is the newest, largest piece of architecture in the app — read this
whole section before changing anything CC-related.

**Why it exists**: the business runs two customer-care teams against the
same client base — "Desk A" and "Desk B" — 10 agents and roughly half the
pandits each, so bookings, agents and pandits are split evenly across both.
A person signed into one desk must never see the other desk's queue. Admin
sees both, always.

**Roster**: `AGENTS` carry `careTeam:'A'|'B'` directly. Pandits inherit it
from their agent (`panditTeam()`). The 6 originally hand-authored agents (and
their 13 pandits) were kept as-is and tagged; 14 more agents (and 5 pandits
each) were generated programmatically in a `growRosterToTwoDesks()` IIFE
right after the hand-authored arrays, to reach 10/10 without hand-typing ~150
records. (Total pandits landed around 83, not exactly 100, because the
original 13 weren't evenly distributed — flagged as an approximation, not a
bug, if you go looking for exactly 100.)

**Booking → desk**: every booking gets `careTeam` at creation (`mkBooking`),
round-robin — **not** derived from whichever agent ends up assigned. This
means desk-scoping works even before a booking has an agent.

**Isolation, enforced twice**:
1. `seatBookings(list)` / `seatAgents(list)` helpers filter to `S.careSeat`
   whenever `S.role==='cc'` (no-op for every other role). Used in `ccQueue`,
   `ccHistory` (`devoteesDirectoryBody`/`filteredDevotees`), `ccSla`,
   `ccCancel`, `ccTemple`, `agentRoster`, `agentOptions`,
   `autoAssignOne`/`autoAssignEligible`, `panditsFor`.
2. `agentAvailability()` **hard-rejects** an agent whose `careTeam` doesn't
   match the booking's `careTeam`, as a second line of defence so even a
   manual override can't cross desks.

**Login — "Log in A" / "Log in B"**: the same `CARE_USERS` credentials work
for either desk; the desk is picked at sign-in, not tied to an account.
Occupancy is a **same-browser, cross-tab lock** — `localStorage['nm_care_seats']`
holds `{A: {userId,name,at}|null, B: {...}|null}`. Claiming/releasing goes
through `claimSeat()`/`releaseSeat()`. A `window.addEventListener('storage', …)`
re-renders other tabs live when a seat changes; `beforeunload` best-effort
releases the seat on tab close/reload.

**Important, honestly-disclosed limitation**: this is a pure client-side app
with **no backend**, so the lock can only coordinate tabs on the *same
browser*. It cannot make a desk "occupied" across two different people's
computers — there is no server to ask. If a real multi-machine lock is
needed, that requires an actual backend (a small API + a real DB or even
just a serverless key-value store), which is outside what this static
prototype can do.

**Admin oversight**: Admin → Care team access → "Both desks, live" shows both
desks' current occupant, since-when, open-booking counts, and a **Force
release** button per desk (for when the automatic release doesn't fire — a
crash, a killed browser). Admin → Agents & pandits shows each agent's desk
with a toggle to move them (pandits move with their agent automatically,
since the desk is derived).

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

## 10. Admin panel — current capabilities

Tabs (`adminScreen()`): Overview, Bookings, **Details of devotees** (same
underlying table as CC's "All history", unmasked here), Temples, Puja
catalogue, Payout rules, **Agents & pandits** (phone numbers, capacity,
active flag, temple coverage, **desk A/B assignment**), **Care team access**
(CC login management + the live desk-occupancy panel from §8), Blackout
calendar, Payments & settlement, Roles & permissions (a read-only rendering
of the `PERMS` table), Settings.

Everything Admin edits (an agent's phone, a pandit's capacity, a booking's
temple) is read live by Customer Care/Agent/Pandit screens from the same
in-memory arrays — there is no separate sync step, because it's the same
objects, not a copy.

## 11. Known gaps / open items in *this* app

- **No login gate on the Admin panel itself.** `adminScreen()` — the
  function, not just a screen — has no auth check. `ADMIN_USERS` exists as a
  data array and the only entry point is one footer link, but anyone who
  reaches `admin.*` gets straight into full control: CC credential
  management, both desks' live data, every booking's full unmasked PII.
  This is the single biggest thing to fix before this is more than a demo.
  The shape of the fix is already scoped (mirror `careLoginScreen()`/
  `ccScreen()`'s gate pattern exactly, with `S.adminUser` instead of
  `S.careUser`) but not built.
- The header search bar (`.dash-search`, "Search bookings, devotees,
  pujas…") is permanently `disabled` — decorative only, never wired up.
- The two-desk seat lock is same-browser only (§8) — a real fix needs a
  backend.
- Pandit count landed at ~83, not exactly 100 (§8) — cosmetic, not
  functional.

## 12. Relationship to the separate `admin-portal` project

**`admin-portal` is a completely different codebase** at
`/Users/suranjeet/admin-portal/`, built and maintained separately by the
user. It is *not* the "Admin panel" role described in §10/§11 above, which
lives entirely inside this repo's `index.html`. Do not assume Claude working
in `admin-portal` has any access to this repo, its `DB`/`AGENTS`/`PANDITS`
arrays, or anything else described in this file — they're different
origins, different processes, and were explicitly kept unsynced by the
user's own choice ("I'll edit that admin portal separately later on"). If
work in `admin-portal` needs to reflect concepts from here (the two-desk
model, the masking rules, the data shapes), **read this file for the
concepts, then re-implement them against `admin-portal`'s own actual code**
— don't assume any file, function or variable name here exists over there
without checking.
