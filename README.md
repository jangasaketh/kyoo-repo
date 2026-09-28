# Kyoo

Live outpatient queue tracking for Hyderabad. Shows the token each doctor is
actually serving right now, and works out the time you should leave your house.

**This is a working prototype with generated sample data.** No real clinics,
doctors, fees or queues are represented, no real money moves. See [Status](#status).

---

## Run it

The site is static and has no build step.

```bash
git clone https://github.com/<you>/kyoo.git
cd kyoo
python3 -m http.server 8080      # or: npx serve .
# open http://localhost:8080
```

It needs a local server rather than opening the file directly, because ES
modules are blocked under `file://`. Any static server will do.

### Publish it

`index.html` is at the root, so GitHub Pages works with no configuration:

Settings → Pages → Source `Deploy from a branch` → Branch `main`, folder `/ (root)`.
`.nojekyll` is present so Jekyll leaves the files alone.

Everything works on Pages — search, booking, refunds, calendar, the dashboards.
Accounts and tokens live in that browser's storage. The optional backend below
moves them to a real database.

---

## What it does

### For patients
- **Live board** of tokens being served near you, across 63 Hyderabad areas
- **Filter by hospital** — pick one or several, or browse hospitals directly
- Search by doctor, hospital, area or symptom (`knee pain` finds orthopaedists)
- 20 specialities; filter by distance, wait, fee, language, gender, punctuality
- **Seen-by time** — queue position plus the journey, not just the in-clinic wait
- **Leave-by time** — the only number that changes what you actually do
- **Live capacity** — how many the doctor has seen, how many wait, how many token
  slots remain against their declared daily cap. Full sessions stop taking bookings
- **Punctuality exposed** per doctor (`usually 28 min late` / `starts on time`)
- Book for yourself or a family member
- **Reschedule or cancel with a full refund up to 1 hour before your turn**
- **Google Calendar and .ics export** with reminders at intervals you choose
- Live alerts as your turn approaches, and when to leave

### For clinics
- **Front desk**: call next, mark no-show, issue walk-in tokens. Every tap
  reaches every watching patient
- **Doctor view**: your own queue, who is waiting, your start delay as patients see it
- **Hospital admin**: every doctor's queue, capacity used, worst start delays, refunds
- **Platform owner**: dwell time, booking funnel, search terms, busiest hospitals

### The demo clock
The slider in the header scrubs the day from 6am to 10pm. Sessions follow real
Hyderabad OP timings — roughly 9–1 and 5–9, shut in between. At 3pm you get the
afternoon-gap state, not a blank page.

---

## Try it in two minutes

1. Open the site and accept or decline the tracking prompt — both work.
2. Sign in with a demo account (the login screen has one button per role;
   password `demo1234`, numbers 9000000001–9000000005).
3. As **Patient**, take a token. Note the leave-by time, then add it to your calendar.
4. Open a second tab, sign in as **Front desk**, and press *Call next patient*.
5. Watch the token move in the first tab. That is the entire product.

---

## How the numbers are produced

All simulated, deterministically seeded so the city looks the same every load.

| Quantity | Method |
|---|---|
| Daily cap | Each doctor declares how many patients a day they will see (8 for a psychiatrist, 60+ for a busy GP). Consult time is derived from it, not the other way round |
| Session cap | The daily cap split across sessions in proportion to their length |
| Tokens issued | ~32% pre-booked and present at open, the rest walking in until the cap is reached |
| Completed | Tokens below the one being served |
| Tokens left | Session cap minus tokens issued. At zero the listing stops accepting bookings |
| Distance | Haversine between coordinates, ×1.34 road factor |
| Travel time | 18 km/h in peak hours (8:30–11:30, 17:00–21:00), 26 km/h otherwise |
| Seen-by | `max(queue reaches you, you arrive)`; if you have already missed it, arrival plus a re-slot penalty |
| Leave-by | Queue-reaches-you minus travel minus a 6 minute buffer |
| Cancellation cutoff | 1 hour before **expected consultation time**, not before booking time — so the window moves with the queue |

---

## Architecture

```
index.html              shell only
src/
  app.js                router, live tick, event wiring
  core/
    seed.js             Hyderabad areas, specialities, name pools
    data.js             hospitals and doctors — generated, or hydrated from the server
    queue.js            clock, geography, the queue and projection engine
    store.js            persistent tables + BroadcastChannel cross-tab sync
    auth.js             five roles, sessions, demo accounts
    appointments.js     booking, the cutoff, reschedule, refund ledger
    calendar.js         Google Calendar links, .ics with VALARM reminders
    analytics.js        consent-gated dwell time, funnel, search terms
    api.js              the seam — local mock, or HTTP to the backend
    sync.js             mirrors server state into the local store
  views/                ui, state, search, hospitals, appointments, account, dashboards
  styles/base.css
server/                 optional Node + SQLite backend (see server/README.md)
test/
  core.test.mjs         68 assertions, pure logic
  server.test.mjs       36 assertions, live HTTP
  ui.test.mjs           61 assertions, headless Chromium against the mock
  integration.test.mjs  14 assertions, the real UI against the real server
```

**The seam.** `src/core/api.js` chooses where data comes from; `src/core/sync.js`
keeps the two worlds in step. With no backend the browser store is the source of
truth. With one, the server's records replace the generated city at boot, every
write goes to the server first, and the result is mirrored back into the local
store — so the views keep their simple synchronous reads and never learn which
mode they are in.

`test/integration.test.mjs` proves this rather than asserting it: it drives the
real UI against the real server and checks that a booking reaches the database,
that the session is an httpOnly cookie the page cannot read, and that a token
booked in one browser profile is visible in another.

---

## Optional backend

The static site is complete on its own. The backend exists for the things a
browser genuinely cannot do honestly: real password hashing, shared state
between devices, and **refunds the client cannot forge**.

```bash
node server/index.js          # no dependencies, Node 22+
# then open http://localhost:8787/?api=1
```

It uses `node:sqlite`, built into Node 22, so there is nothing to install and
nothing to compile. See [server/README.md](server/README.md).

The refund rule is enforced there, from the server's own clock and its own copy
of the queue. A client asking for a refund it is not owed is refused — there is
a test that sends `{refund: 999999}` on a ₹700 token and checks the server pays
what it should.

Frontend and backend share one clock deliberately (`src/core/queue.js`). If the
browser believed it was 11am and the server believed it was 3am, they would
compute different queue positions for the same token, and the cutoff would be
measured against a time the patient never saw.

---

## Tests

```bash
npm install playwright --no-save   # once, for the browser suites
npm test                           # all four
```

179 assertions. The browser suites drive a real Chromium — jsdom cannot run ES
modules, so a real browser is the only honest way to test this.

They cover the things that are easy to claim and hard to get right: cross-tab
sync (two pages, a desk action in one, the token moving in the other), the
refund rule under both branches of the cutoff, and role guards.

Tests pin the clock rather than depending on when they run — a suite that only
passes between 9 and 1 is not a suite. `KYOO_CLOCK=660` does it server-side; the
demo slider does it in the browser.

---

## Privacy

Analytics record **nothing** until consent is given, and revoking consent
deletes what was collected. Any user can see, download or delete everything held
about them from their account page, with no support ticket. Under the DPDP Act
2023 that is the floor, not a feature — queue position tied to a person is
arguably health data.

---

## Status

Prototype. Not connected to any real clinic, and not a medical service.

Before this could go live:

- Real clinic onboarding, and desk software people will actually use
- OTP sign-in rather than passwords, which is what Indian users expect
- A payment gateway (Razorpay or similar) behind the ledger. The refund logic
  and the ledger are already shaped for it; only the gateway calls are missing
- Google Distance Matrix for real traffic, replacing the speed estimate
- A DPDP Act compliance review, and a decision on data residency
- Clinic verification, so listings cannot be gamed

The product is the easy half. Getting clinics to press the button is the whole
game — give the desk software away, ship a waiting-room display first, and go
one locality deep before going city-wide. Queue data is worthless at 5% coverage
and indispensable at 60% in one area.

## Licence

MIT. See [LICENSE](LICENSE).
