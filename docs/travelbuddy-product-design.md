# TravelBuddy — Product & Solution Design

*Travel with a trusted friend.*

**Status:** Draft v0.1 for HITL review. Input to `PRD.md`, `Context.MD`, ADRs and the Roadmap. It is not a Spec or Plan.
**Source brief note:** the AI-Powered Features section of the brief contains only *Smart Delay Detection* (a trailing empty heading followed it). Anything beyond that is marked *proposed*.

---

## 1. Product thesis

Alarm apps know the clock. Location-sharing apps know where you are. Neither knows **where you are going**. TravelBuddy is destination-aware: it measures progress along a journey and acts on it. It wakes you before your stop, tells the people who care that you are on your way, and tells them when you have arrived.

**Mission:** make every journey safer, smarter and more connected.

**Differentiators**

1. Alarms keyed to *arrival*, not to a clock time.
2. Contacts watch from a plain web link: no account, no app install.
3. Privacy as a product feature: per-journey sharing, expiry, instant revoke, end-to-end encryption.

**North-star metric:** *Safe arrivals confirmed per week* (journeys that ended with the alarm firing on time and, where shared, the contact notified).
**Guardrail metrics:** missed-stop rate, alarm-fire reliability, battery drain per journey hour, share-link revocation latency.

## 2. Personas and jobs-to-be-done

| Persona | Job | Must-have | Design implication |
|---|---|---|---|
| Daily commuter | "Don't let me miss my stop." | Fast start, recurring routes, reliable alarm | Saved routes, one-tap "start usual journey" |
| Business traveller | "Let colleagues and family know where I am." | Multi-contact sharing, ETA accuracy | Contact groups, delay-aware ETA |
| Student | "Parents shouldn't have to ask." | Auto-share, auto arrival message | Guardian presets, low-battery alert |
| Elderly traveller | "Reassure me and them." | Big type, few taps, check-ins | Simple mode, large-target UI, optional carer-set-up |
| Solo traveller | "Help if something goes wrong." | SOS, check-ins, last known location | SOS reachable in ≤ 2 taps from any screen |

The **trusted contact** (a wife, parent, colleague) is a first-class user with their own experience, even though they have no account.

## 3. Primary journey: Raja, Edinburgh → London King's Cross

1. **Set destination.** Search "King's Cross" and pick *London King's Cross (station)*.
2. **Set alert.** Choose "10 min before arrival".
3. **Share.** Pick wife from contacts; the app generates a secure link and sends it by WhatsApp or SMS.
4. **Start.** The share message goes out: "Raja has started travelling to London."
5. **En route.** The traveller dashboard shows progress, ETA and countdown; the contact page shows the live map.
6. **T-10 min.** A full-screen alarm plays and a push notification fires. The contact is told "approximately 10 minutes away".
7. **Arrival.** The app auto-detects arrival, or Raja taps *I've arrived*. The contact gets "Raja has arrived safely at London King's Cross."
8. **End.** "Journey completed. Live tracking has ended." The link expires and the location data is retained or purged per the user's retention setting.

## 4. Scope and release slicing

| Release | Theme | Contents |
|---|---|---|
| **MVP (0.1)** | "Never miss your stop" | Destination search; arrival alarm (5/10/15/30/custom); live tracking and progress; traveller dashboard; manual arrival confirm |
| **0.2** | "Let them watch" | Secure link share (web viewer, no account); journey started / near / arrived / ended notifications; revoke and expiry; hide-exact-location |
| **0.3** | "Keep me safe" | SOS; check-ins; low-battery alerts; SMS / WhatsApp / email / QR sharing; E2EE hardening |
| **0.4** | "Smart" | Delay detection and ETA adjustment; saved places; recurring routes; auto-arrival |
| **1.0** | Production | GDPR tooling (export/erase), retention controls, role-based access, accessibility audit, store release |

Rationale: the alarm is the core promise and the riskiest technically (background location), so it ships first and gets proven before sharing is layered on.

## 5. Functional design

### 5.1 Smart Arrival Alarm

- **Triggers:** remaining time ≤ N minutes, using the live ETA. Preset intervals are 5, 10, 15 and 30 minutes, plus a custom value.
- **Failsafes (the alarm must never silently fail):**
  - A **distance geofence** fires if the ETA source is unavailable.
  - A **scheduled time fallback** is set from the planned arrival time.
  - A **"tracking lost" warning** fires if no location fix arrives for N minutes.
- **Alarm behaviour:** a full-screen intent, bypassing silent/Do-Not-Disturb where the OS allows it (iOS Critical Alerts need an Apple entitlement, so apply early). Escalating volume. Snooze is optional and limited. Dismissal is explicit.
- **Destinations:** cities, stations, airports, bus stops, postcodes, landmarks, saved places, dropped pins. Stations and airports get a **destination radius** (e.g. 300 m) rather than a point, so "arrived" is meaningful.
- **Edge cases:** a destination reached before the alarm fires (skip the alarm and go to the arrival flow); a route change mid-journey; tunnels and GPS loss (dead-reckon on last speed, flag low confidence); an overshoot (user passes the destination without stopping → "You may have missed your stop").

### 5.2 Live journey tracking

Tracks current position, route progress %, ETA, remaining distance, speed and route changes.

- **Map:** position, destination, planned route polyline, and a travelled/remaining split.
- **Sampling:** adaptive. Higher frequency near the destination or at low speed. Lower frequency on long steady stretches to save battery.
- **Route source:** for rail, the timetable/real-time feed plus the track geometry. For road, a routing engine. For unknown modes, fall back to great-circle progress plus recent speed.
- **Travel mode** (train / bus / car / walk / other) is chosen or inferred, because it determines the ETA model.

### 5.3 Trusted companion sharing

- **Channels:** secure link, SMS, WhatsApp, email, QR. They all carry the same link.
- **Viewer:** a responsive web page. Live map, progress, ETA, remaining distance, status, last-updated timestamp, optional battery.
- **Per-share controls:**
  - What is shared (location, ETA, battery).
  - **Exact vs approximate location** (snap to ~1 km grid or to the route segment).
  - **Expiry** (at arrival + X, or fixed time).
  - **Instant revoke.**
- **Viewer alerts:** a no-account viewer can opt into **Web Push** from the page (or leave a number for SMS where the traveller consents). Note iOS Web Push needs the page added to the home screen.

### 5.4 Safe-arrival notifications

Four templated events. Wording is fixed and localisable:

| Event | Text |
|---|---|
| Started | "{name} has started travelling to {destination city}." |
| Near | "{name} is approximately {n} minutes away from {destination city}." |
| Arrived | "{name} has arrived safely at {destination}." |
| Ended | "Journey completed. Live tracking has ended." |

### 5.5 Family safety

- **SOS:** one tap, with a 3-second cancel window to prevent false triggers. It sends current location and an alert to the emergency contacts, upgrades the share to high-frequency live tracking, and offers the "call emergency services" dialer. TravelBuddy supplements emergency services and does not replace them. The app must say this plainly.
- **Check-ins:** "Are you OK?" at a configurable interval, with a response window (e.g. 2 min). A missed response escalates: a second prompt, then a contact alert with the last known location.
- **Low battery:** thresholds are configurable (e.g. 20% / 10% / 5%). Contacts are told that location updates may stop and are given the last known position.

### 5.6 Dashboards

- **Traveller:** destination, ETA, distance remaining, current location, alarm countdown, sharing status, battery, progress %.
- **Contact:** location, live map, ETA, progress, remaining distance, arrival status, last update. It shows an explicit **staleness banner** ("Last updated 4 min ago") so a quiet phone is not mistaken for a safe one.

### 5.7 AI-powered: Smart Delay Detection

**Detects:** traffic delays, train delays, route changes. **Effect:** automatic ETA adjustment, with the alarm trigger and contact notifications following the new ETA.

Approach, layered by cost and risk:

1. **Deterministic signals first.** Real-time rail feeds (e.g. National Rail / Darwin in the UK), road traffic APIs, and plain observed-speed-versus-plan comparison.
2. **Statistical ETA correction.** Blend the planned ETA with the observed progress rate: ETA = now + remaining distance ÷ smoothed expected speed on the remaining segments.
3. **ML only where it earns its keep.** Examples are learning stop-dwell times, or predicting that a late train will recover time. Introduce it after there is data and a baseline to beat.
4. **Route-change detection.** Deviation from the corridor beyond a threshold for a sustained period triggers a re-route and a re-computed ETA. The user is notified: "Your route has changed; ETA now 14:32."
5. **Explain it.** Every ETA change carries a plain reason ("Delay reported at Peterborough, +12 min").

**Principle:** the alarm must stay safe if the AI is wrong. Delay logic may move an alarm *later* only when a trusted source confirms the delay, and the distance geofence always remains as a backstop. Delay information is never the only thing standing between a user and a missed stop.

*Proposed, not in the brief:* conversational assistant ("when do I arrive?"), anomaly detection for unexpected long stops (feeds check-ins), and natural-language destination search.

## 6. UX architecture

**Principles:** glanceable (readable in 2 seconds on a train), calm (a companion, not a dashboard), one-handed, safe by default, accessible.

**Navigation (traveller app):** a bottom tab bar. Tabs are *Journey* (home), *Places*, *Contacts*, *Settings*. SOS is a persistent affordance during a live journey.

**Key screens**

1. Onboarding: value in one screen, permissions requested *in context* (location → "always" only when starting the first journey, with a plain explanation; notifications; contacts).
2. Home / New journey: destination search, recent and saved places.
3. Journey setup: mode, alert interval, share toggle, review → *Start*.
4. Live journey: map with progress rail, big ETA, countdown ring, share status chip, SOS.
5. Alarm: full screen, huge "Wake up, you're 10 min from King's Cross".
6. Arrival: "You've arrived", confirm, share-ended summary.
7. Sharing manager: active links, recipients, revoke, expiry.
8. Contact web viewer: map-first, status header, staleness banner, no sign-up wall.
9. Safety settings: check-in cadence, emergency contacts, battery thresholds.
10. Privacy centre: retention, export, delete.

**Accessibility:** WCAG 2.2 AA. Dynamic type. Voiceover/TalkBack. Never colour-only status. An elderly "Simple mode" (larger targets, fewer options, optional carer-assisted setup).

**Tone:** warm, concise, first-name. The notification wording in §5.4 is the voice reference.

## 7. Solution architecture

### 7.1 Key design decision: device-led, server-relayed

The risky promise ("wake me up") must **not** depend on the network or on a server. So:

- **On device:** location capture, ETA/progress computation, alarm scheduling and firing, event detection (started / near / arrived / check-in due).
- **On server:** relay, fan-out, durable state for viewers, notification dispatch, delay-data enrichment, and revocation enforcement.

This also makes the E2EE story coherent: the server relays ciphertext and does not need to understand location.

### 7.2 Components

```
 Traveller app (iOS/Android)                Cloud                       Contact
┌──────────────────────────┐   TLS   ┌──────────────────────┐   TLS   ┌──────────────┐
│ Location engine          │────────▶│ API gateway / Auth   │◀────────│ Web viewer   │
│ ETA + progress engine    │         │ Journey service      │         │ (static SPA) │
│ Alarm scheduler (local)  │         │ Share service        │         │ decrypts in  │
│ Event detector           │         │ Realtime relay (WS)  │────────▶│ browser      │
│ Crypto (E2EE)            │         │ Notification service │         └──────────────┘
│ Local store              │         │ Delay-intel service  │
└──────────────────────────┘         │ Geo/route adapters   │──▶ Maps, routing, rail & traffic feeds
                                     │ Data store + cache   │──▶ Push (APNs/FCM/WebPush), SMS, email
                                     └──────────────────────┘
```

| Component | Responsibility |
|---|---|
| Mobile app | Cross-platform with native modules for background location, geofencing and alarm. React Native or Flutter are both viable. The native pieces are the hard part, whichever is chosen. |
| Web viewer | Static SPA on a CDN. Decrypts payloads client-side. Works without login. |
| API gateway / auth | Traveller identity (passwordless/OIDC). Viewers get **capability tokens**, not accounts. |
| Journey service | Journey lifecycle (planned → active → arrived/ended/cancelled), and settings. |
| Share service | Creates share links, scopes, expiry, revocation. |
| Realtime relay | WebSocket/SSE fan-out of encrypted location updates to viewers. |
| Notification service | Templated, rate-limited dispatch over push, SMS, WhatsApp, email. Idempotent. |
| Delay-intel service | Ingests rail/traffic feeds, answers "is this journey delayed?". |
| Geo adapters | Geocoding, routing, station and rail data behind an interface, so providers are swappable. |

Suggested stack (a recommendation to confirm via `/factory-init-tech-context`): TypeScript end to end; Postgres + PostGIS and Redis; a managed realtime layer; Azure-hosted to match Enate's estate. Treat this as a proposal, since the repo currently has no declared stack.

### 7.3 Data model (core)

- **User**: id, display name, settings, retention preference.
- **Place**: id, user, label, geometry, radius.
- **Journey**: id, user, destination (place/geometry/radius), mode, planned route, alert offsets, status, timestamps, retention-until.
- **LocationSample**: journey, time, position, accuracy, speed, battery. *Stored encrypted and subject to retention.*
- **Share**: id, journey, scope flags (location / ETA / battery), precision, expiry, revoked-at, key-wrap metadata.
- **ViewerSession**: share, anonymous id, push subscription (optional).
- **Event**: journey, type (started / near / arrived / ended / sos / checkin-missed / low-battery), time, delivery status.
- **EmergencyContact / TrustedContact**: name, channel, consent record.
- **AuditLog**: access, revoke and export actions.

### 7.4 Privacy and security design

**E2EE for no-account viewers.** The traveller's device generates a random journey key. The share link carries it in the **URL fragment** (`https://share.travelbuddy.app/j/<token>#<key>`). Fragments are never sent to servers. The viewer page uses the key to decrypt updates, while the server stores and relays ciphertext. Revoke = the server stops serving the token and the traveller can rotate the key.

**Honest trade-off to decide (ADR needed):** server-computed things (server-sent "10 minutes away" SMS to a contact who has no app, server-side delay enrichment tied to a specific position) need *some* plaintext. The proposed split:

- The **device** decides when an event occurs and sends a minimal templated event (type + destination label + minutes). The server sees only that.
- Precise coordinates remain E2EE and are readable only by the viewer page.
- Delay enrichment is done by the *device* querying the delay-intel service with the route/service identifier (e.g. a train service ID), not with raw coordinates.

**Other controls**

- Link tokens: ≥128-bit random, single-journey, expiring, revocable, rate-limited. Revocation propagates in seconds (short-TTL relay channels, so a cached page stops updating).
- Role-based access: *Owner* (traveller), *Viewer* (capability token), *Support/Admin* (no access to journey content; audit-logged for metadata). Optional *Guardian* role where a parent or carer can co-manage a student's or elderly user's settings, with the traveller's consent.
- Transport TLS 1.2+, encryption at rest, secrets in a managed vault (never in code or logs), location data excluded from logs.
- **GDPR:** lawful basis = consent for sharing and for background location. Data minimisation. Purpose limitation. Defined retention (default: raw location purged within 24 h of journey end; the user can choose shorter or longer). Self-serve export and erasure. DPIA before launch (continuous location tracking is high-risk processing). Contact-consent: the traveller is the controller for the details they enter about others; a consent record is kept, and the first message to a contact carries opt-out.
- Abuse: the product can be misused for covert tracking. Mitigations: the traveller always sees an active share indicator; a persistent notification shows while sharing; no "silent" tracking of another person; a viewer cannot start tracking a traveller.

### 7.5 Reliability and platform realities

- **Background location:** iOS ("Always" authorisation, significant-change plus region monitoring) and Android (foreground service, battery-optimisation exemptions) are the dominant technical risk. Spike this first.
- **Alarm delivery:** local scheduling and full-screen intents (Android), Critical Alerts / Time-Sensitive notifications (iOS). Test on aggressive OEM battery managers.
- **Offline:** the app keeps working with no signal (alarm and progress are local). Updates are queued and synced. The viewer shows staleness.
- **Fail-safe stance:** an ambiguous or missing signal never produces a false "arrived safely". Arrival is declared only on confirmed geofence entry or explicit user confirmation; "no data" shows as *stale/unknown*, never as safe.
- **Scale shape:** location updates are the load. At 10k concurrent journeys sampling every 10 s, that is about 1k writes per second, which is modest. Design for fan-out per journey (few viewers each) rather than global broadcast.

## 8. Notifications and messaging

| Channel | Used for | Notes |
|---|---|---|
| Local alarm | Arrival alarm | Device only; independent of network |
| Push (APNs/FCM) | Traveller prompts, check-ins | |
| Web Push | Opt-in viewer alerts | iOS requires home-screen install |
| SMS / WhatsApp / email | Share invitation; event messages for viewers without push | WhatsApp business templates need approval; cost per SMS needs modelling |

## 9. Risks and open questions

| # | Risk / question | Mitigation / next step |
|---|---|---|
| 1 | Background location killed by OS | Early spike; user guidance; "tracking lost" alerts |
| 2 | iOS Critical Alerts entitlement refused | Apply early; fallback to Time-Sensitive plus loud in-app alarm |
| 3 | E2EE vs server-side notifications | ADR on the device-led event model (§7.4) |
| 4 | Rail/traffic data licensing and coverage | Pick launch geography (UK first, given the brief); confirm provider terms |
| 5 | SMS/WhatsApp cost per journey | Model unit economics; prefer link and push for repeat contacts |
| 6 | Covert-tracking misuse | Visible indicators; abuse policy; reporting |
| 7 | False SOS / check-in fatigue | Cancel window; adaptive prompts |
| 8 | Regulatory: DPIA, children's data (students under 18) | Legal review before the student/guardian features ship |
| 9 | Liability if an alarm fails | Clear terms; reliability testing; never imply a guarantee |
| 10 | Monetisation | Proposal: free core alarm + 1 contact; paid for unlimited contacts, SOS escalation, history. To validate. |

## 10. Quality strategy

Aligned with the Factory three-tier testing model and ratchet.

- **Tier 1 (unit and integration):** ETA/progress maths, event detector state machine, alarm-trigger rules, token and crypto handling, notification templating and idempotency.
- **Tier 2/3 (end to end and field):** a replayable journey simulator (recorded GPX/timetable traces, including tunnels, delays and route changes); real-device alarm tests across iOS/Android versions and OEM battery managers; real train runs for release candidates.
- **Non-functional:** battery budget (target ≤ 5% per hour of tracking, to be measured), revocation latency (< 5 s), alarm accuracy (fires within ±1 min of the true T-N), accessibility audit, penetration test on the share/relay surface.

## 11. Suggested Roadmap Features (for `/factory-roadmap`)

1. Journey core: destination search, journey model, progress and ETA engine.
2. Arrival alarm and failsafes (includes the background-location spike).
3. Traveller dashboard and live map.
4. Secure sharing and web viewer (E2EE).
5. Safe-arrival notification pipeline.
6. SOS, check-ins, low-battery protection.
7. Smart Delay Detection.
8. Privacy centre and GDPR tooling.
9. Saved places and recurring routes.
10. Accessibility and Simple mode.

## 12. Decisions needed from a human before Specs

1. Launch geography (drives rail and map data providers).
2. Mobile framework (React Native vs Flutter vs native).
3. The device-led vs server-led event model (§7.4), recorded as an ADR.
4. Monetisation direction.
5. Whether guardian/child accounts are in scope for v1 (legal impact).
