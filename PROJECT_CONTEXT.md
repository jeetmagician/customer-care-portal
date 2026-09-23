# Namonamaha Customer Care Portal — A to Z

> Read this first. It is written so a fresh Claude session (no memory of earlier work) or a new
> developer can understand exactly what this app is, what it does today, how it is built, what was
> tried and removed, and what still belongs in the **separate `admin-portal` project**.

---

## A. What it is

**Namonamaha** is a temple-puja booking service: a devotee books a puja (online, or in person at a
temple), an onsite **agent** meets them at the temple, a **pandit** performs the rite. This repo is a
**single-file prototype of the internal Customer Care portal** for that service — a CRM where customer
care staff confirm bookings, chase balance payments, assign agents/pandits, handle escalations and
cancellations, and review history.

It is a **demo**, not production: no backend, in-memory data (persisted to browser `localStorage`),
simulated payments/WhatsApp/calls, plaintext demo passwords. The UI says "Customer care demo".

## B. Quick start

```
cd customer-care-portal
python3 -m http.server 8000      # then open http://localhost:8000
```
Sign in with **`admin` / `Admin@123`**. Hard-refresh (Cmd+Shift+R) after code changes. Repo:
`https://github.com/jeetmagician/customer-care-portal` (`main`).

## C. Files

| File | Purpose |
|---|---|
| `index.html` | The whole app (~350KB): inline CSS + one inline `<script>`, vanilla JS, no build, no deps |
| `logo-transparent.png` | Header/sidebar/login logo (background removed; referenced as `?v=2` to bust cache) |
| `logo.jpg`, `logo-new-source.jpg` | Original logo sources (not referenced by the page) |
| `wallpaper.jpg` | Cosmic background behind the portal |
| `PROJECT_CONTEXT.md` | This document |

## D. Login (the only entry)

One page: **Login ID + Password** (eye icon toggles visibility; Enter submits).
- `admin` / `Admin@123` (`ADMIN_USERS`) → straight into the Customer Care portal.
- Customer-care staff logins (`CARE_USERS`, seeded one per temple, e.g. `priya.care` /
  `Kamakhya@101`, `rahul.care` / `Bagala@102`, `asha.care` / `Ugratara@104`,
  `bikash.care` / `Navagraha@105`, `manas.care` / `Lankeshwar@106`, `ratul.care` /
  `Jagannath@107`, `dipa.care` / `Kalighat@108`; `nibedita.care` is inactive) also work and open
  the **same** portal — there is no per-temple or per-user restriction any more.
- Wrong credentials → "Incorrect login ID or password." Log out clears the session.
- The check is client-side against plaintext demo passwords (not real security).

## E. The Customer Care portal — every screen

After login you see **all temples' clients together** (temple is shown on each booking). Sidebar:

1. **Work queue** — stat cards (Total client bookings, Running now, Escalations, Needs a call, Needs
   balance link, Balance outstanding, Fully paid/no agent) and bookings grouped by *what is blocking
   them*: Poojas running now → Escalated (handle first) → Awaiting the confirmation call → Confirmed:
   send the balance link → Balance outstanding → Photographs pending → In progress and closed. Rows
   show client/pooja IDs, temple, package, dates, assigned agent/pandit, status pills, SLA-overdue
   note, and buttons (Edit booking, Send/Resend completion code). Includes **Auto assign agent**.
2. **All history** — "Details of devotees": one filterable table with columns Devotee, Pooja, Agent,
   Pandit, Pooja rate, Advance payment, Due, Full payment, Total, Date & time. Filters: booking type,
   payment status, search (name/phone/booking/client ID, matched on the real data), pooja date range,
   A–Z letter strip. **Export CSV** has its own independent date range. Fixed-width columns
   (`<colgroup>`), names truncate with ellipsis; a pulsing dot marks a running pooja.
3. **SLA & escalations** — the SLA table (below), bookings currently inside an SLA, and the
   escalation reason codes.
4. **Cancellations** — refund policy table and cancelled bookings with refund %.
5. **Message log** — the registered message templates (WhatsApp/Email/SMS) and when they fire.

**Booking detail** (click a client): booking-at-a-glance, the step panel for the current stage
(Step 1 confirmation call with editable package/date/time/agent, auto-written + AI-drafted + manual
call notes; Step 2 send balance link; outstanding-balance follow-ups; "assign a pandit myself"),
agent availability roster, pandits for the temple, devotee card, money, assignment (agent/pandit
contact, completion code, arrival code), SLA, call log, append-only **audit trail**, progress
timeline, reminder schedule. **Edit booking** popup changes temple, package, date, time, city, email,
language, agent and wish with a mandatory reason (audited; price change may need extra payment/refund).
**Temple staffing** and **Agent pooja flow** popups show agents, their pandit teams and workload, with
assign/reassign/unassign.

**Auto assign agent** (`autoAssignOne`): for fully paid bookings with no agent, picks the covering,
available agent (active, in shift, no clash) with the fewest running bookings, preferring agents who
have a free pandit — i.e. even distribution.

## F. Devotee privacy (applies to everyone in this portal, admin included)

- Names: first 3 letters of each word + `****` → `Sur**** Cho****` (`maskName`).
- Phones: country code + first 4 digits + `*****` → `+91 9000*****` (`maskPhoneDisplay`).
- Devotee contact is **Internet call only** (simulated masked line, `devoteeContactCC`) — no direct
  `tel:` Call and no WhatsApp to devotees. Agent and pandit contact stay real (Call / Internet call /
  WhatsApp icons via `contactIconBtns`).
- Masked everywhere: lists, detail pages, history table, CSV export, AI call notes, auto-assign panel,
  and the global **Activity** feed (`maskEventText`, render-time only; stored events keep real names).
- Search still matches the real underlying name/phone (only the display is masked).
- `PERMS['booking.devotee.name'/'phone']` are `care:'masked'`.

## G. Data model (all top-level `let` globals, seeded by `seed()`)

- **`TEMPLES`** (8): T1 Kamakhya, T2 Bagalamukhi, T3 Umananda, T4 Ugratara, T5 Navagraha, T6 Lankeshwar
  Shiv Mandir (all Guwahati, Assam), T7 Jagannath (Puri, Odisha), T8 Kalighat (Kolkata, West Bengal).
  Fields: id, name, city, state, live flag, hours, meeting point, modes. T7/T8 are "Onboarding".
- **`PUJAS`** (20): `P1–P12` in-person, `O1–O3` online, `S1–S5` subscription — price, lead days, payout
  split (pujari/samagri/agent…).
- **`AGENTS`** (20; 6 hand-written + 14 generated by `growRoster()`): id, name, phone, temples covered,
  active flag, shift string. **`PANDITS`** (~83): id, name, phone, temple, `agentIds`, skills, daily
  capacity, leave flags.
- **`CARE_USERS`**, **`ADMIN_USERS`** (logins, §D), **`BLACKOUTS`** (temple closure dates that block
  bookings and flag existing ones).
- **`DB`** = `{settings, bookings, events}`. Seed: 19 bookings across temples T1–T6, 35 events.
- A **booking** (`mkBooking`): ids (`NMH-####`, client `CL-####`, pooja `PJ-####`), `devotee` {name,
  gotra, phone, email, city, dob, pob, wish}, temple/puja/mode, money (`amount`, `advanceAmt`,
  `balanceAmt`, paid flags, `payments`), `status`, agent/pandit, OTP + arrival code, `callLog`,
  `timeline`, `audit`, `reminders`, `escalated`, `cancellation`, `feedback`, `invoice`, `settlement`.

## H. Booking lifecycle and rules

`DRAFT → ADVANCE_PAID → CC_CONFIRMED → LINK_SENT → FULLY_PAID (Awaiting pandit) → PANDIT_ASSIGNED →
COMPLETED → SETTLED`, plus `CANCELLED` and an `escalated` flag. Modes: **In-person** (agent + arrival
code), **Subscription** (weekly/monthly, agent), **Online** (no agent, pandit assigned directly).
Advance is 20% (settings), GST 18%.

**SLAs** (`SLAS`): first confirmation call within 4h of advance (breach → red row); balance link within
1h of the call; pandit assigned by T-24h (auto-escalates to care); photographs within 6h of completion;
payout within 72h; refund within 120h.

**Refund tiers** (`REFUND_TIERS`): ≥30 days 100%; 7–30 days 85%; 3–7 days 50%; 24–72h 25%; <24h 0%
(one free reschedule may be offered). **Escalation reasons** (`TRANSFER_REASONS`) e.g. no pandit free,
pandit absent, temple closed, agent unavailable.

**Reminder ladder** (`LADDER`): T-30d, T-15d, T-7d, T-3d, T-48h (OTP), T-24h (agent details), T-3h.
**Templates** (`TEMPLATES`): booking created, receipts, balance link, OTP, reminders, assignment,
escalation, media, feedback, refund, etc.

**Permissions** (`PERMS`, `can(role,path)` → edit/read/lock/masked/hide/flow, rendered by `field()`).
Care can edit temple, package, date, agent, email/city/dob/pob/wish/prasad; name/phone are masked;
amount is locked; status moves only via workflow.

## I. Demo controls (top bar)

**Simulated clock** with **+1 day**; **Reset** reseeds the demo; **Desktop/Mobile** preview toggle
(only used by the mobile-style roles); floating **Activity** drawer with the simulated outbound message
feed (Push/WhatsApp/Email/Payment/System). Payments (Razorpay), WhatsApp, calls and AI voice are all
**simulated** and labelled so.

## J. Architecture

One `<script>` ordered CONFIG → DATA → STORE → UI → SCREENS → ROUTER.
- **STORE**: every mutation goes through action functions (`payAdvance`, `assignAgent`,
  `reassignAgent`, `unassignAgent`, `assignPandit`, `escalate`, `cancelBooking`, `setField`…) which write
  the audit trail and event feed together.
- **Router**: one global state `S` (role, sub-tab, modal, drafts, login state), one `render()` doing a
  full `innerHTML` replace of `#app`, one delegated click listener dispatching on `data-act`, plus
  `change`/`input`/`keydown` listeners. Live-search uses a targeted patch (`patchDevList`) so the input
  keeps focus.
- Gotcha: any action that calls `render()` wipes un-stored form input; capture fields into `S` first
  (the login form does this for the password toggle).

**Persistence**: `DB`, `TEMPLES`, `PUJAS`, `AGENTS`, `PANDITS`, `CARE_USERS`, `ADMIN_USERS`,
`BLACKOUTS` are saved to `localStorage['namonamaha-care-demo-v15']` (`STORAGE_KEY`) on every render.
`seed()` always runs first, then `restoreDemo()` overwrites it with saved data — so **bump the version
suffix whenever any of those arrays' shape or seed data changes**, or browsers keep replaying stale data.
`S` is never persisted.

## K. What was built, then removed (do not reintroduce unless asked)

The user simplified step by step. Built and later removed: Desk A/B split with "Log in A/B" and a
same-browser seat lock; a "Select temple location" state picker with per-state admins; an Admin panel
(temple list, per-temple sign-in pop-up, masked credential editor, add temple, create CC login, change
own login); per-temple/zone isolation of bookings and the Activity feed; hiding the role tabs. Their
code is partly left in `index.html` as reference and is **unreachable**: `adminScreen`, `adCareTeam`,
`adDash`, `adBookings`, `adDevotees`, `adPeople`, `adPayments`, `templeLoginModal`, the Devotee
(`userScreen`), Agent and Pandit mobile screens. The top role tabs are hidden for everyone
(`#roles` is empty).

## L. Known limitations

- No backend: data lives per browser; login is a client-side plaintext check.
- Anyone can read plaintext demo passwords from source/localStorage.
- The header search box is decorative (disabled).
- Pandit count is ~83 (generated), not a round 100.
- Simulated integrations (payments, WhatsApp, masked calling, AI voice).

## M. Testing approach used

There is no test suite. Changes were verified by (1) extracting the inline script and running
`node --check`, (2) running it in a Node `vm` sandbox with stubbed `document`/`localStorage` and calling
`ccScreen()`/action handlers to assert behaviour, and (3) headless Chrome screenshots
(`chrome --headless --screenshot`) for visual checks.

## N. What to build in the separate `admin-portal`

`admin-portal` (`/Users/suranjeet/admin-portal/`) is a **completely different codebase** — do not assume
any file, function or data here exists there, and it shares no data with this app (different origin,
no backend). Read this document for concepts, then implement against its real code. Suggested scope:
- **Real admin authentication** (hashed passwords, sessions).
- **Temple management**: add/edit/deactivate temples across states, search + state filter.
- **Customer-care login management**: create/reset/deactivate logins, hashed passwords, unique IDs;
  changes must reach this app → needs a **shared backend/database** (temples, bookings, agents,
  pandits, CC users).
- **Per-temple / per-state access & isolation and "open temple CC"** (prototyped here, then removed):
  build on the shared backend with a signed, short-lived handoff link into this app.
- **Admin screens** (reference code in `index.html`): Overview, Bookings, Details of devotees
  (unmasked, export), Puja catalogue, Payout rules, Agents & pandits, Blackout calendar, Payments &
  settlement, Roles & permissions, Settings.
- Keep this app's rule: devotee data masked in customer care.

## O. Deploy

Push to GitHub (`git push` on `main`). The app is static, so any static host works; there is no build.
