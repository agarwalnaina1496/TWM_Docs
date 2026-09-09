# Booking Handoff — Providers & Deep Links

How Travel With Me turns a frozen itinerary into booking actions. TWM never
holds, verifies, or confirms a booking — it hands the traveller a search on a
partner site, prefilled with whatever the itinerary already knows, and gets
out of the way.

Sources of the decisions below: TWM-129/130/131 (trusted-action contract and
provider research), TWM-194/197 (MVP affiliate-redirect model), TWM-216
(stay provider capability matrix), TWM-195 (transport gateway-leg scope),
TWM-196 (flight redirect → Aviasales), TWM-205 (activities out of scope).
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

This varies by domain, and **only stay has a real per-partner capability
model** today:

- **Flight** — a live cached price when TWM can resolve one, plus an
  Aviasales search-form redirect. The Aviasales format is confirmed, so
  origin, destination, dates, and passengers genuinely prefill.
- **Stay** — a three-tier capability model (below), chosen per partner
  based on whether that partner's deep-link format is confirmed.
- **Train and bus** — no capability model. The link opens the partner
  (ixigo) carrying TWM's own generically-named parameters, which ixigo's
  public pages do not necessarily read. In practice the traveller lands on
  ixigo and searches from scratch. A confirmed ixigo train/bus deep-link
  format is future work.

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
| **Agoda** | Known-destination search | "Search Agoda" | Agoda's search needs an internal city ID, not a place name. TWM holds that metadata for **Goa only** right now — so the Agoda card appears for Goa and is **hidden for every other destination** (no fabricated search). Expanding the metadata table is the way to widen this. |
| **ixigo** | Destination redirect | "Browse ixigo hotels" | Always resolves. India-native, trusted domestic brand, no foreign-tourist markup, and its own affiliate account. ixigo's hotel deep-link parameters were never confirmed, so it opens the destination hotel *listing* only — dates and guests are set on ixigo. |

So for a typical non-Goa trip the stay drawer shows **two cards** (Booking.com
+ ixigo).

**Decided in research but not shipped** (TWM-131 named five stay partners;
TWM-216 kept three). Before TWM-216, every stay partner used a generic
*guessed* search path. TWM-216 kept only the partners whose *native* URL
shape could be confirmed and dropped the rest rather than ship guessed links:

- **Hotellook** — was intended as the *primary* stay partner: a Travelpayouts
  meta-search that already compares Booking.com / Agoda / others behind one
  link, covering the most ground with the least UI clutter. Its base domain
  is still wired; re-add once its search URL format is confirmed.
- **Hostelworld** — the budget / hostel tier. Separate affiliate signup
  (Partnerize), not done. Base domain wired.

**Rejected outright:**

- **Trip.com** — weak India-domestic hotel coverage for the current market.
- **Airbnb** — no affiliate programme (Airbnb Associates ended March 2021),
  regardless of older mockups that referenced it.

### Known stay gaps

- **Agoda is Goa-only** — see the table. Every other destination shows no
  Agoda card.
- **ixigo carries nothing** — destination listing page only, by the
  "don't guess parameters" rule.
- **Booking.com resolves the destination itself** — TWM sends the place as a
  free-text search string, not Booking.com's internal destination ID, so
  Booking.com fuzzy-matches it. For a small town with one property it can land
  directly on that property rather than an area search. Pinning it to a
  city / region needs a Booking.com destination-ID table (the same shape as
  Agoda's).
- **No flexible-date support** — for a month-precision trip TWM sends no dates
  at all; Booking.com's own "I'm flexible" month view could be prefilled
  instead once that URL format is confirmed.

## Transport

A "Transport options" action appears on the **two gateway legs only** — the
leg that brings the traveller into the trip and the leg that takes them out
(TWM-195 V1 scope). Internal city-to-city legs are shown in the itinerary for
context with no booking action. Each feasible mode is one card; the drawer
marks a recommended mode by fixed priority (flight > drive > train > bus),
which is a routing heuristic, not a price or quality claim.

| Mode | Partner | What the traveller gets | Why this partner |
|---|---|---|---|
| **Flight** | Aviasales | A **live cached price** when TWM can resolve one (the CHECK_PRICES path), plus an Aviasales search-form redirect. The Aviasales format is confirmed, so origin, destination, dates, and passengers genuinely prefill. | Aviasales is TWM's integrated flight partner — same Travelpayouts account as the live-price path. |
| **Train** | ixigo | A redirect to ixigo. The traveller enters the route and date on ixigo. | IRCTC-authorised partner, India-native, no foreign-tourist markup. IRCTC has no usable public API; unofficial scrapers are legally grey and unreliable. 12Go was dropped — real IRCTC bookings but ~30% over direct price and built for foreign tourists. |
| **Bus** | ixigo | A redirect to ixigo, same as train. | Same reasoning as train. |
| **Drive** | — | Distance and estimated duration, computed by TWM. No booking action. | Nothing to book — the traveller's own vehicle or a cab arranged locally. |

Train and bus redirects are **not prefilled** — the link carries TWM's own
generic parameter names, which ixigo's pages don't necessarily read. Flight
is the exception. There is no per-partner capability model for transport the
way there is for stay.

**Decided in research, superseded or unshipped:**

- **ixigo as a flight redirect** — TWM-131 approved ixigo as a second flight
  option. TWM-196 replaced it with the Aviasales search-form redirect (same
  Travelpayouts account as the live price), so ixigo is no longer a flight
  partner.
- **redBus for buses** — verified affiliate programme (EarnKaro, ~₹150 per
  booking). It is in the bus allowlist and its base domain is wired, but the
  UI only ever asks for a mode, never a specific bus partner, so ixigo is
  always the one chosen. Surfacing redBus as its own card is a follow-up.

**Rejected outright:**

- **MakeMyTrip** — carried over from an early mockup, never actually
  researched.
- **12Go** (train) — see the Train row.

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
disclosure shown. Two separate affiliate relationships:

- **Travelpayouts** — one marker. The network nominally covers Aviasales,
  Hotellook, Booking.com, and Agoda, but tracking is **wired only for
  Aviasales and Hotellook** (the shapes confirmed during research). It is the
  same account the live flight-price path uses.
- **ixigo** — its own account via EarnKaro / Cuelinks, covering flights,
  trains, buses, and hotels. Separate signup, separate ID.
- **redBus, Hostelworld** — affiliate programmes exist; no tracking wired.

So **Booking.com and Agoda links currently carry no tracking** — the
relationship is available but the tracked-link format isn't wired, and the
links stay honest in the meantime.

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
