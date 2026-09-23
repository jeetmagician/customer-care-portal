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

1. **One login page** (`index.html`, `localhost:8000`). First pick **"Select temple location"**
   (state/UT from `INDIA_STATES`), then ID + password; a login only works for its own
   location. Admins are **zone admins** (`ADMIN_USERS[].state`): `admin` / `Admin@123`
   = Assam, `odisha.admin` / `Odisha@Admin1`, `wb.admin` / `WestBengal@Admin1`.
   Temple customer-care staff use the same form (their temple's state must match).
2. **Admin** lands on one page, **Temple customer care**, showing only their state's
   temples (list, pop-up and Activity feed are zone-scoped; other states are invisible).
   From it an admin can:
   - **Change any customer-care login ID / password / temple / active flag** — shown masked
     (`pr*****` / `Ka*****`) until *Show / change* is clicked.
   - **Add a temple** in their own state (created as "Onboarding", `live:false`).
   - **Create a customer-care login** for any temple in their state (login IDs are unique
     across customer care *and* admin logins).
   - **Change their own admin ID / password.**
   - Click a temple → **sign-in pop-up** accepting only that temple's own credentials
     (no admin bypass) → that temple's portal, with a "Back to admin" button.
3. **Temple customer care** (distinct password per temple, e.g. Kamakhya `priya.care` /
   `Kamakhya@101`): a CRM scoped to that temple only — Work queue, All history (table +
   CSV export), SLA & escalations, Cancellations, Message log, auto-assign agent, edit
   booking, confirm call, send balance link / OTP. Isolation covers bookings, the Activity
   feed, and the role tabs (hidden for everyone; the Devotee/Agent/Pandit screens are
   unreachable because they list every booking).
4. **Devotee privacy in CC**: names `Sur**** Cho****`, phones `+91 9000*****`, Internet-call
   only (no direct Call / WhatsApp to devotees). Agent/pandit contact stays real.
5. All data is in-memory + `localStorage` (`namonamaha-care-demo-v13`). No backend, so
   logins are a client-side check against plaintext demo passwords.

## What to build in the separate `admin-portal` (suggested scope)

This app prototypes the admin side of temple customer care; the real `admin-portal`
should own it properly. Build there:
- **Real authentication** for admins (hashed passwords, sessions, per-state/zone roles) —
  here it is a plaintext demo check.
- **Temple management**: add/edit/deactivate temples in any state, with a **search box and
  state filter** (100s of temples). Adding a *state/zone* and its admin is a job for a
  super-admin in admin-portal (here zone admins are hard-coded in `ADMIN_USERS`).
- **Customer-care login management**: create/reset/deactivate one login per temple; store
  passwords **hashed**, prefer *reset* over *reveal*; enforce unique IDs.
- **"Open temple CC"** as a real handoff: signed, short-lived token/link from admin-portal
  into this app, plus the per-temple sign-in this app already does. Needs a shared backend.
- A **shared backend/database** (temples, bookings, agents, pandits, CC users) so both apps
  read the same data — today each has its own in-memory copy.
- Admin screens this app dropped from its sidebar that belong in admin-portal: Overview,
  Bookings, Details of devotees (unmasked, export), Puja catalogue, Payout rules, Agents &
  pandits (roster, phones, capacity, coverage), Blackout calendar, Payments & settlement,
  Roles & permissions, Settings. Reference code still in `index.html`: `adDash`,
  `adBookings`, `adDevotees`, `adPeople`, `adPayments`, …
- Keep this app's **rules**: zone/temple isolation, devotee masking in CC, admin sees real data.

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
`localStorage` under a versioned key, currently `namonamaha-care-demo-v13`
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

The top role tabs are hidden for everyone (`#roles` is always empty), so no one can reach
the Devotee/Agent/Pandit screens, which list every booking.

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

## 8. Login flow and temple-wise customer care (details)

**One login form** (`careLoginScreen`) is the entry point. The user first picks a
state/UT (`#clst`, `S.stateDraft`); `cc-login` then checks `ADMIN_USERS` first, then
`CARE_USERS`, and the account's state must equal the picked one (an admin's own
`state`, or the state of a customer-care user's temple). Otherwise: "Incorrect login ID
or password for <state>".
- **Zone admin** → the Admin panel, a single tab "Temple customer care" (`adCareTeam`),
  gated by `S.adminUser`. `templesInZone()` limits everything to that admin's state;
  `ad-open-cc`, `ad-save-careuser`, `ad-add-temple`, `ad-add-careuser` all re-check the zone.
  The other admin screen functions (`adDash`, `adBookings`, …) remain as reference code only.
- **Temple staff** (`CARE_USERS[].templeId`) → straight into their own temple's portal.

**Temple-wise**: each temple has its own CC login(s) with different passwords (seeded
`Kamakhya@101`, `Bagala@102`, …). A CC session is scoped by `ccTempleId()` / `scopeBk()`
(queue, All history, SLAs, cancellations, auto-assign) and `visibleEvents()` (Activity feed).

**Admin opening a temple's portal**: "Open <temple> customer care →" (`ad-open-cc`) shows a
sign-in pop-up (`templeLoginModal`, `S.templeLogin`) accepting only that temple's
credentials (`ad-temple-login`); on success the portal opens with `S.fromAdmin=true` and a
"← Back to admin" button (`cc-back-admin`). No admin bypass.

**New temples** (`ad-add-temple`) are pushed to `TEMPLES` as `live:false`, hours
"Onboarding" (the same shape as the seeded T7/T8), ids `T<n+1>`. Login IDs are unique across
`CARE_USERS` and `ADMIN_USERS` (`loginTaken()`).

History: a Desk A/B split with a seat lock was built then removed by the user; don't
reintroduce it. Roster: 20 agents, ~83 pandits (`growRoster()`).

The separate `admin-portal` project can't do this itself (different origin, no shared
data); it needs a backend with real credentials/tokens (see the TL;DR).

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
(`maskCred`, `S.revealCred`); plus add-temple, create-CC-login and change-own-login forms. The other admin screens' code
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
