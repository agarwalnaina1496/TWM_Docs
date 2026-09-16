# Booking Handoff — Providers & Deep Links

How Travel With Me turns a frozen itinerary into booking actions. TWM never
holds, verifies, or confirms a booking — it hands the traveller a search on a
partner site, prefilled with whatever the itinerary already knows, and gets
out of the way.

Sources of the decisions below: TWM-129/130/131 (trusted-action contract and
provider research), TWM-194/197 (MVP affiliate-redirect model), TWM-216
(stay provider capability matrix), TWM-195 (transport gateway-leg scope),
TWM-196 (flight redirect → Aviasales), TWM-205 (activities out of scope),
TWM-230 Increment 2 (transport provider capability model — real per-partner
deep links for train/bus, replacing the single generic ixigo redirect),
TWM-230 Increment 2c (ixigo reinstated as a second flight SEARCH_REDIRECT
partner, its own confirmed deep link).
This document is the canonical record; the Linear issues are the working
history.

## The model

Every booking action is one of:

| Action | What it is | Where it's used |
|---|---|---|
| **CHECK_PRICES** | TWM's own resolved live/cached data (real normalized offers). | flight only |
| **SEARCH_REDIRECT** | A link to a search on a partner site, prefilled where possible. No price, rating, availability, or "booked" claim ever attached. | flight, train, bus, stay |
| **PROVIDER** | A partner handoff carrying a TWM-resolved price. | none yet — reserved for when a hotel/rail inventory integration lands. Not valid for flight. |

Only flight has a live-data path today (Aviasales, via Travelpayouts).
Trains, buses, and stays are SEARCH_REDIRECT only — no train/bus/hotel
inventory API was both usable and self-service during provider research, so
TWM presents these honestly as search handoffs rather than implying it has
schedules or availability.

### How prefilled a redirect actually is

This varies by domain. Stay, transport, and flight now all have a real
per-partner capability model:

- **Flight** — a live cached price when TWM can resolve one (Aviasales,
  CHECK_PRICES), plus **two independent** SEARCH_REDIRECT options —
  Aviasales and ixigo (TWM-230 Increment 2c) — each with its own confirmed
  deep-link shape. Both genuinely prefill *only when both endpoints resolve
  to IATA airport codes*; a route TWM can't resolve to real airports
  degrades each to a plain destination search, never a fabricated prefill
  claim.
- **Stay** — a three-tier capability model (below), chosen per partner
  based on whether that partner's deep-link format is confirmed.
- **Train and bus** — a two-tier capability model (below, TWM-230 Increment
  2), chosen per partner based on whether the route/date resolve to that
  partner's confirmed deep-link shape.

The single rule behind every gap below: where a partner's public deep-link
format was **not confirmed** during research, TWM does not guess parameter
names — it drops to a plainer redirect (or hides the partner) rather than
ship a link that might silently break.

### Stay capability tiers

| Tier | Meaning |
|---|---|
| **Prefilled search** | Destination + dates + party land on the partner's own search, ready to run. |
| **Destination search** | The partner opens on the right place; the traveller picks dates there. |
| **Destination redirect** | The partner opens on a destination listing page; dates *and* guests are chosen on the partner site. |

### Train & bus capability tiers

| Tier | Meaning |
|---|---|
| **Prefilled search** | Origin, destination, and date land on the partner's own route search, ready to run. |
| **Destination search** | The partner opens on a generic search surface; the traveller enters the route and date there. |

## Stay

A "Stay options" action appears on **every stay segment** in the itinerary.
Each approved partner that resolves is one card in the drawer; a partner that
can't resolve for this destination is silently absent. **No "Our pick" badge
is ever shown on a stay card.**

The drawer also shows Atlas's non-binding per-night price band (budget /
mid-range / premium) as context — that is an Atlas estimate, clearly labelled
non-binding, not a provider price.

| Partner | Tier | CTA | Notes / why |
|---|---|---|---|
| **Booking.com** | Prefilled search *(dates known)* / Destination search *(no dates)* | "Search Booking.com" | Always resolves. Native search URL is confirmed, so destination, check-in / check-out, traveller count, currency (INR), and room count (fixed at 1) prefill when known. Without dates it still opens on the destination. |
| **ixigo** | Destination redirect | "Browse ixigo hotels" | Always resolves. India-native, trusted domestic brand, no foreign-tourist markup, and its own affiliate account. ixigo's hotel deep-link parameters were never confirmed, so it opens the destination hotel *listing* only — dates and guests are set on ixigo. |

So the stay drawer shows **two cards** (Booking.com + ixigo) for every
destination.

**Rejected outright** (TWM-131 named five stay partners; TWM-216 kept three
whose native URL shape could be confirmed, dropped the rest rather than ship
guessed links; Hotellook, Hostelworld, and Agoda were fully removed from the
partner allowlist, base domains included, TWM-230 Increment 2b):

- **Hotellook** — was intended as the *primary* stay partner: a Travelpayouts
  meta-search that already compares Booking.com / Agoda / others behind one
  link. Its native search URL format was never confirmed.
- **Hostelworld** — the budget / hostel tier. Affiliate signup (Partnerize)
  was never completed.
- **Agoda** — its search needs an internal numeric city ID, not a place
  name. TWM held that metadata for a single hand-curated entry, Goa, and
  hid the card everywhere else — the exact hardcoded place→ID pattern
  this story rejected elsewhere (the original `_IXIGO_STATION_CODES`
  table, ixigo-as-a-bus-partner). Revisit only with a real city-ID
  resolution mechanism (e.g. if Travelpayouts' own deep-link generator
  resolves a destination automatically — see Affiliate & tracking).
- **Trip.com** — weak India-domestic hotel coverage for the current market.
- **Airbnb** — no affiliate programme (Airbnb Associates ended March 2021),
  regardless of older mockups that referenced it.

### Known stay gaps

- **ixigo carries nothing** — destination listing page only, by the
  "don't guess parameters" rule.
- **Booking.com resolves the destination itself** — TWM sends the place as a
  free-text search string, not Booking.com's internal destination ID, so
  Booking.com fuzzy-matches it. For a small town with one property it can land
  directly on that property rather than an area search. Pinning it to a
  city / region needs a Booking.com destination-ID table — the same shape of
  problem that got Agoda dropped, so not pursued without a real resolution
  mechanism.
- **No flexible-date support** — for a month-precision trip TWM sends no dates
  at all; Booking.com's own "I'm flexible" month view could be prefilled
  instead once that URL format is confirmed.

## Transport

A "Transport options" action appears on the **two gateway legs only** — the
leg that brings the traveller into the trip and the leg that takes them out
(TWM-195 V1 scope). Internal city-to-city legs are shown in the itinerary for
context with no booking action (a booking drawer for a *bookable* internal
leg, where a gateway hub coincides with a planned stop, is TWM-230 Increment
3 — not yet shipped). Each feasible mode gets its own card grid — one card
per approved partner for that mode, at stay-drawer parity (TWM-230 Increment
2). There is no "recommended mode" or "Our pick" claim on any transport
card — modes and partners are presented as equal options.

| Mode | Partner(s) | What the traveller gets | Why this partner |
|---|---|---|---|
| **Flight** | Aviasales + ixigo | Aviasales: a **live cached price** when TWM can resolve one (the CHECK_PRICES path), plus a **prefilled search-results link** (`aviasales.com/search/{IATA}{DDMM}{IATA}[{DDMM}]{passengers}`, browser-verified TWM-230 Increment 2d) when both airports resolve; otherwise a dateless pre-filled *search form* (`aviasales.com/?params=...`) or the bare homepage. ixigo: a **second, independent** prefilled search (`ixigo.com/search/result/flight?from=...&to=...&date=DDMMYYYY&...`, browser-verified TWM-230 Increment 2c) under the same airport-resolution condition; otherwise the ixigo flights landing page. | Aviasales is TWM's integrated live-price partner (same Travelpayouts account as the live-price path). ixigo has its own confirmed deep link and its own EarnKaro affiliate account — a real second option, not a live-price path. |
| **Train** | ixigo | A **prefilled search** (`ixigo.com/trains/search-pwa/from/{code}/to/{code}/{date}`) when both stations resolve to a real IRCTC-style station code and a date is known; otherwise a plain ixigo trains search. Station codes are resolved from a bundled ~8,700-row open dataset (`twm/services/station_resolution/`, CC0-licensed `datameet/railways` data), never a hand-typed place → code table. | IRCTC-authorised partner, India-native, no foreign-tourist markup. IRCTC itself has no documented deep link (a stateful single-page app) and was dropped as a partner. |
| **Bus** | redBus | A **prefilled search** (`redbus.in/bus-tickets/{from}-to-{to}?onward={date}`) when origin and destination are known; the onward date is added when known. | redBus's route/date URL shape was browser-verified during TWM-230 Increment 2 research — the only bus partner with a confirmed public deep link. It has a confirmed EarnKaro affiliate programme, but tracking is not yet wired (see Affiliate & tracking). |
| **Drive** | — | Distance and estimated duration, computed by TWM. No booking action. | Nothing to book — the traveller's own vehicle or a cab arranged locally. |

**Decided in research, superseded or unshipped:**

- **ixigo as a flight redirect** — TWM-131 approved ixigo as a second flight
  option; TWM-196 replaced it with the Aviasales-only search-form redirect
  (same Travelpayouts account as the live price) because a confirmed ixigo
  flight deep link hadn't been researched at the time. TWM-230 Increment 2c
  found and browser-verified one (see the table above) and reinstated
  ixigo as a second SEARCH_REDIRECT alternative — the live CHECK_PRICES
  price stays Aviasales-only, unchanged.
- **ixigo as a train/bus redirect (pre-TWM-230-Increment-2)** — the original
  MVP sent every train and bus request to ixigo carrying TWM's own
  generically-named, unread parameters (route/date entered from scratch on
  ixigo). Increment 2 replaced this with the confirmed per-partner shapes
  above; ixigo is no longer used for bus at all (see next point).
- **ixigo for buses** — researched during Increment 2 and dropped: ixigo's
  bus search needs an internal numeric city ID per city, not a plain place
  name, so a name-only deep link isn't possible without building and
  maintaining that ID table — out of scope for this increment.
- **Aviasales' own deep-link shape, corrected (TWM-230 Increment 2d)** —
  the shape this story originally implemented
  (`search.aviasales.com/flights/?origin_iata=...`), from an older
  Travelpayouts support article, was never browser-tested against the
  live site until a full re-verification pass this session — it drops
  every param and redirects to the `.ru` marketing homepage. The current
  shape (table above) was confirmed against Travelpayouts' present
  "Aviasales affiliate links" article and browser-verified directly.
  Every other deep link in this document (ixigo train/flight/stay,
  redBus, Booking.com) was re-verified live in the same pass and
  confirmed genuinely working — this was the one gap.

**Rejected outright:**

- **MakeMyTrip** — carried over from an early mockup, never actually
  researched.
- **12Go** (train) — real IRCTC bookings but ~30% over direct price and
  built for foreign tourists.

## Activities & tickets

**Explicitly out of scope for MVP** (TWM-205, decided 2026-08-26). Atlas
flags activities that need advance booking (e.g. a safari with limited daily
slots) via a `booking_readiness` signal, and the itinerary shows that as a
small badge on the item. There is **no booking action of any kind** — no
partner, no redirect, not even a "mark as noted" toggle (considered and
rejected as a false sense of tracking). Revisit once a real activity /
ticket partner is evaluated.

## Affiliate & tracking

A redirect link carries a tracking parameter only when a real tracking ID is
configured; otherwise it is a plain, honest link with no affiliate
disclosure shown. Two separate affiliate relationships.

**Two different wiring shapes matter here (TWM-230 Increment 2b research)** —
confusing them is why "just wire the tracking param" undersold the real
work for half these partners:

- **Shape A — direct query-param.** The network hands you a static ID; you
  append it yourself as a URL param on the partner's own site. Self-serve,
  code-only, no outbound call needed.
- **Shape B — link-wrapping.** The network requires submitting your target
  URL *to them* (their own tool, API, or redirect domain) and they hand back
  a tracked link. Appending a param to the partner's own URL yourself does
  **not** earn commission here — confirmed against Travelpayouts' and
  EarnKaro's own documentation.

| Partner | Programme | Shape | Wired? |
|---|---|---|---|
| Aviasales | Travelpayouts | A (`marker=`) | ✅ |
| ixigo | EarnKaro / Cuelinks | A (`affiliate_id=`) | ✅ |
| Booking.com | Travelpayouts | **B** — Partner Links API / `tp.media` redirect, or a per-account static `aid` if one is confirmed on the real dashboard | ❌ not wired |
| redBus | EarnKaro | **B** — "paste your link, get a profit link" wrapper; no plain static param confirmed | ❌ not wired |

(Agoda was dropped from the product entirely, TWM-230 Increment 2b — see
Stay above — so it's not carried in this table anymore. Travelpayouts also
covers it via the same Shape-B mechanism as Booking.com, plus a confirmed
**1-day cookie window**, if it's ever revisited.)

Confirming Booking.com/redBus's exact Shape-B mechanism needs the real
Travelpayouts and EarnKaro account dashboards — public documentation
describes the shape but not account-specific specifics (the Partner Links
API endpoint, whether a static `aid` is issued, EarnKaro's link-generation
API if any). If it turns out to require a live wrapping call, that is a
small new integration (a link-wrapping step at request time), not a query-
param change — its own scoped increment.

So **Booking.com links currently carry no tracking** — the relationship is
available but the tracked-link format isn't wired, and the links stay
honest in the meantime.

Tracking IDs are supplied only through environment configuration — never
committed, never typed into a chat.

## What TWM never does

- Hold or verify a booking, or show a "confirmed" / booking-reference state.
- Attach a price, rating, star count, or availability claim to a
  SEARCH_REDIRECT.
- Guess a partner's deep-link parameters. An unconfirmed format means a
  plainer redirect, a lower stay tier, or a hidden partner — never a
  fabricated link.
- Let an agent generate a booking link. Every link comes from the
  deterministic backend resolver.
