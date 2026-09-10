# WhatsUpNext — Specification

WhatsUpNext turns an Apple TV into a conference room display: walk up and
instantly see whether the room's free, what's coming up next, and who's in
the current meeting. It plugs into whatever calendar system you already
use, and doubles as a supplementary display for building emergency
notifications (lockdown alerts, etc.), integrating with systems like
Singlewire InformaCast.

This document covers **the App** (the tvOS client) and **the Service**
(the self-hosted admin backend) — the two components in this repository.
WhatsUpNext also ships as a hosted SaaS at whatsupnext.io; that control
plane is a separate, closed-source system and is not covered here. The
Service itself has zero awareness of which distribution mode it's running
under — no tenancy, billing, or SaaS-related code exists in it at all.

## Goals & non-goals

- **Goal**: the App keeps working even if the Service is unreachable or
  down — it owns its own calendar polling/caching and holds its own
  calendar credentials, rather than depending on the Service for live
  data.
- **Goal**: minimal component count. Prefer folding functionality into an
  existing model/service over introducing a new one; prefer relying on
  tooling already in the stack (Kamal, Rails 8 defaults) over adding new
  infrastructure.
- **Goal**: the Service is deployable anywhere reachable — no dependency
  on a specific cloud platform.
- **Non-goal**: the App/Service pair is not the primary emergency
  notification system for a building — see Emergency Notifications below.
  It's a supplementary channel, not a life-safety-critical one.
- **Non-goal**: no multi-tenancy at the Service level, ever. One Service
  instance = one organization, always.

## Naming conventions

- **"the App"** — the tvOS client running on the Apple TV in the room.
- **"the Service"** — the Rails backend (admin UI + pairing/config API).
  Deliberately has no concept of being "self-hosted" vs. "SaaS-hosted" —
  that distinction is irrelevant to the Service and exists only outside
  it, in how/where an instance happens to be deployed. No naming or code
  branches on it.
- **the Pairing Code flow** — manual device registration: the App shows a
  code, an admin enters it in the Service's UI.
- **the Zero-Touch flow** — MDM-driven registration via Managed App
  Configuration, no human interaction on the App.

## Architecture

### Deployment requirements

- **Decision: public reachability is a requirement of running the
  Service, for every deployment.** A self-hosted install must point a
  domain at the Service with inbound HTTPS reachable from the internet;
  this is a stated installation/operation requirement, not an assumption
  to design around. This isn't really a new burden — Kamal's built-in
  proxy already needs public reachability on ports 80/443 to provision
  Let's Encrypt certificates, so every Kamal-deployed instance was
  implicitly going to need this anyway. Practical effect: the Service can
  always receive inbound webhooks (calendar provider push, emergency
  notification provider push) — there is no NAT'd/unreachable-Service
  case to design around. (This doesn't remove the value of the App's own
  resilience — Option A for calendar OAuth and the App's direct
  emergency-provider fallback, both below, are about surviving a *Service
  outage*, which can still happen to a reachable Service, not about
  reachability itself.)
- **Considered and rejected: a centralized relay (e.g. a Lambda) that
  receives all provider webhooks and pushes APNs directly to the App**,
  bypassing the need for the Service to be reachable at all. Technically
  cleaner for reachability specifically, but rejected: it's a new
  permanent component — compute, tenant/device-token routing, its own
  uptime — sitting in the critical path of every deployment. Conflicts
  with the minimal-component-count goal above. Don't revisit this without
  a concrete reason the reachability requirement has become a real
  problem in practice.

### The Service

- **Backend**: Ruby on Rails 8
- **Datastore**: SQLite + Solid Queue / Solid Cache / Solid Cable (Rails 8
  defaults) — no Postgres/Redis dependency, keeps the Service trivially
  self-hostable (single file + Docker)
- **CSS**: Tailwind
- **Deploy mechanism**: Kamal (Docker + SSH to any host) — host-agnostic
  by design
- **Tenancy**: single-tenant per running instance, always. No `Account`
  model, no billing code.

### Authentication (to the Service's admin UI)

- **Mechanism**: OmniAuth + a generic **OpenID Connect** strategy (not
  provider-specific gems) — one integration covers Google Workspace,
  Microsoft Entra ID, Okta, Auth0, etc. Each deployed instance configures
  **one OIDC provider** via env vars (client id/secret/issuer URL), plus
  `SERVICE_HOST` (the Service's own public host[:port]) — the OIDC
  strategy needs it to construct the callback `redirect_uri`, since
  `omniauth_openid_connect` requires that value explicitly and won't
  derive it from the request.
- **Provisioning**: **JIT (just-in-time)** — a `User` record is created
  automatically on first successful OIDC login. No allowlist, no
  pre-created users.
- **Authorization**: **none, intentionally.** Access control is entirely
  delegated to the customer's IDP — if their IDP lets someone authenticate
  against the Service's OIDC client, they're in.
- Replaces Rails 8's built-in password-based auth generator. Session model
  concept is kept; `password_digest` is not used.
- **Fail fast at boot if OIDC isn't configured.** `OIDC_ISSUER`/
  `OIDC_CLIENT_ID`/`OIDC_CLIENT_SECRET` are read via `ENV.fetch`, not
  plain `ENV[]` — a deploy missing any of them refuses to boot with a
  clear error, rather than booting successfully and only failing later,
  confusingly, the first time someone tries to sign in. An unconfigured
  instance is an ops/deploy problem to fix directly (set the env vars,
  redeploy), not something the app should work around.
- **Decision: break-glass/bootstrap access, scoped to OIDC configuration
  only, for one specific failure: OIDC is configured but the IDP is
  unreachable/broken.** A single emergency credential — not a `User`
  row, not self-service — exists independent of whether the app can
  currently reach the IDP, so an admin gets back in even when the
  provider itself is down. It unlocks *only* a read-only OIDC settings
  status screen (configured issuer/client id, never the secret) —
  useful for confirming the deployed config looks right while
  diagnosing an IDP-side outage; it does not let you edit OIDC settings
  through the app, since `omniauth_openid_connect` only reads its
  config once at boot from `ENV`, matching its own documented usage —
  changing settings still means editing `ENV` and redeploying, same as
  initial setup. Nothing else in the admin UI (rooms, buildings,
  devices, other configuration) is reachable through it. Keeps a
  leaked/brute-forced break-glass credential's blast radius to "can
  view OIDC status," not "has full admin access" — matching the actual
  failure this path exists to fix. No password-reset flow, no email:
  the credential lives in ENV like other deploy-time secrets (see
  `service/CLAUDE.md`) and is rotated by changing the env var, not
  through the app.

### MDM / device management (of the App, on the Apple TV)

- **Conference Room Display mode (Apple's built-in tvOS feature) is NOT
  used.** It's a full-screen system takeover (AirPlay landing page: wifi
  info, room name, custom message) that replaces the Home Screen — it does
  not host third-party apps, and per MDM vendor docs it **overrides Single
  App Mode** if both are configured on a device. It's a different, Apple-
  native alternative to what we're building, not something we layer the
  App on top of.
- **Kiosk lockdown uses MDM Single App Mode** (ideally Autonomous Single
  App Mode via supervision/Automated Device Enrollment) to pin the App in
  the foreground and prevent exiting to the Home Screen.
- **Managed App Configuration** (MDM pushes server URL + registration
  token to the App via `com.apple.configuration.managed`, enabling
  zero-touch registration) is a standard Apple platform capability, but
  **support for pushing it to tvOS apps varies by MDM vendor** — confirmed
  supported on Jamf (and apparently SimpleMDM); confirmed *not* supported
  for Apple TV on Hexnode. Cannot assume every customer's MDM supports
  this.
- **Decision: support both pairing paths, with a defined precedence.** On
  launch, the App checks for Managed App Configuration first; if present,
  it self-registers with no human interaction. If absent — no MDM, or an
  MDM that doesn't push tvOS app config — it falls back to the manual
  pairing-code screen. Every device works via pairing code regardless of
  MDM; MDM-managed devices get zero-touch as an upgrade on top of that,
  not a replacement for it.
- **AirPlay works alongside Single App Mode by default.** SAM and AirPlay
  are independent MDM controls — AirPlay is only blocked if an admin
  separately enables the "Disable AirPlay" restriction (supervised devices
  only). So end users can still AirPlay a laptop to the room's Apple TV
  even while the App is locked in via SAM.
- **Design implication**: an incoming AirPlay session backgrounds/
  interrupts the App while it's active, and tvOS returns to the App when
  the session ends. The App must refresh room/calendar data on returning
  to the foreground (`scenePhase == .active`), not just rely on its
  polling timer having kept running.
- **Decision: Zero-Touch token model — one shared deployment-wide secret,
  not per-device tokens.** A single `EnrollmentSecret` (rotatable, stored
  on the Service) proves a given `mdm_register` call is legitimately from
  the org's MDM. It doesn't need to be per-device because device
  *identity* is handled separately: **apps can't read their own hardware
  serial number** on tvOS/iOS (blocked since iOS 7, for privacy), so the
  App has no way to self-report which physical device it is. The standard
  workaround: most MDM vendors (Jamf, Mosyle, etc.) support variable
  substitution in Managed App Configuration payloads (e.g. Jamf's
  `$SERIALNUMBER$`), letting the MDM inject the real serial number into
  the config before pushing — so one config profile can be templated
  across an entire fleet while each device still ends up uniquely
  identified.
- **Managed App Configuration payload (concrete keys the App reads)**:
  - `ServerURL` — the App has no built-in way to know its Service's
    address otherwise
  - `EnrollmentSecret` — the shared deployment secret
  - `DeviceIdentifier` — MDM-injected serial number, becomes our
    `mdm_device_id`
- **Decision: pre-provisioning flow.** An admin can create a `Device` row
  ahead of time with `mdm_device_id` + `room_id` already set — e.g. via a
  bulk import mapping serial numbers (sourced from Apple Business
  Manager's device list) to rooms before the hardware even ships. This is
  what makes "order 50 Apple TVs, they arrive and just work" actually
  true.
- **Decision: fallback for unrecognized devices.** If `mdm_register` is
  called with an `mdm_device_id` that has no matching pre-provisioned
  `Device`, don't reject it — auto-create a paired-but-room-unassigned
  `Device`. Graceful degradation to "admin assigns the room afterward via
  the admin UI" rather than requiring perfect pre-provisioning discipline.

### Calendar syncing

- **The App owns polling and local caching of calendar data** — not the
  Service. This is a deliberate scope-limiting choice: the Service stays
  minimal, and the App keeps working even if the Service is down.
- **The Service's role in calendar data is narrow**: receive provider
  webhooks (Google/Microsoft push notifications) and, on receipt, send a
  silent APNs push to the paired App to trigger an immediate poll. tvOS
  does support silent/background push (`content-available`), though with
  tighter background-execution constraints than iOS — not a practical
  problem here since the App is foregrounded almost all the time
  (SAM-locked kiosk); the main use is waking it promptly after an AirPlay
  interruption.
- **The App always keeps its own baseline polling loop**, independent of
  webhook nudges. A webhook/push never arriving (ICS source with no push
  mechanism at all, Service outage) just means the App falls back to its
  own interval.
- **ICS sources are fully autonomous by design**: the App hits the ICS URL
  directly, no Service involvement in the data path at all, ever.
- **Decision: Google/Microsoft OAuth credential handling — Option A.** The
  App holds the OAuth refresh token itself and refreshes its own access
  token directly against Google/Microsoft. Fully autonomous indefinitely,
  regardless of Service uptime. Tradeoff accepted: a live calendar
  credential lives on physical hardware in the room, mitigated by Keychain
  storage + per-device revocation from the admin UI if a TV is
  lost/stolen.
  - Rejected: **Option B** (Service brokers OAuth, hands the App
    short-lived access tokens) — a Service outage breaks live calendar
    updates after ~1 hour (the access token's lifetime), only a bounded
    grace window rather than true independence.
- **Sync mechanism baseline**: polling + local caching, not a
  live/streaming architecture — webhooks are a latency optimization
  layered on top, never the sole mechanism.
- **Decision: fetch/cache window and poll frequency.**
  - **Window**: start-of-today (room-local midnight) through end-of-
    tomorrow — a rolling ~48 hours. Re-fetched in full on every poll (no
    incremental/delta sync). Starting at midnight (not "now") is required
    so an in-progress meeting that started earlier today still correctly
    reads as "current"; extending through tomorrow avoids a gap right at
    midnight rollover and lets the display fall back to "next meeting:
    tomorrow at 9am" when today has nothing left.
  - **Baseline poll interval**: 1 minute, hardcoded default. For ICS
    sources, which have no push capability and depend entirely on this
    number, max staleness of 60s feels immediate; for Google/Microsoft,
    webhook nudges usually beat it. Add small random per-device jitter so
    many rooms in one building don't poll in a synchronized burst.
- **Decision: recurring events and timezone handling.**
  - **Recurring events**: no manual RRULE expansion for Google or
    Microsoft — both APIs expand recurring instances server-side for a
    bounded time range (Google `events.list` with `singleEvents=true`,
    Microsoft `calendarView`). **ICS is the only case requiring manual
    expansion**, bounded to the ~48h fetch window. Must correctly handle
    `EXDATE` (a cancelled instance) and `RECURRENCE-ID` overrides (a
    moved/retitled instance). Use an existing RFC 5545-compliant
    recurrence library rather than hand-rolling RRULE math.
  - **Timezones**: store everything in UTC; convert to the room's
    configured IANA timezone (e.g. `America/Chicago`, never a fixed
    offset) only at display time. ICS recurrence expansion must evaluate
    each occurrence in the event's own timezone, not fixed-UTC-offset
    arithmetic, or recurring meetings silently drift an hour across DST
    boundaries.
  - **All-day events**: on a room's calendar, treated as marking the room
    busy for the entire day — a room resource calendar's all-day event is
    more likely a genuine full-day reservation than the non-blocking
    "reminder" all-day events often mean on a personal calendar.
    Represented as a floating date tied to the room's local "day," not a
    UTC instant.
- **Decision: the App shows a staleness indicator on-device too, not
  just the admin UI.** The whole product exists to answer "is this room
  free" at a glance — confidently displaying room status from a stale
  local cache with no indication could genuinely mislead someone into
  double-booking or standing outside a "free" room that's actually
  occupied. The App tracks its own last-successful-poll timestamp
  locally and, past a staleness threshold, shows a visual "may be
  outdated" indicator alongside the (still-displayed) cached room
  status, rather than either hiding the data or showing it with
  unwarranted confidence — the cached data stays useful and displayed,
  consistent with the App's own resilience goal of working through a
  Service or provider outage, it just isn't presented as equally
  trustworthy as a fresh read. **Threshold: 12 hours** — deliberately
  much longer than the Service-side admin threshold (1 hour, see
  `CalendarSource` notes in Data model): comfortably absorbs an
  overnight outage or a bad afternoon without flapping a warning on the
  actual room display in front of end users, while still bounding the
  worst case (a genuinely broken calendar sitting unflagged on-screen)
  to half a day rather than indefinitely.

### Calendar credential security

- **The threat model worth naming explicitly**: `CalendarSource.config`
  (the OAuth refresh token) is `encrypts`-protected at rest, but the
  Service actively decrypts and transmits it to the App on every
  `credential_version` mismatch — it isn't just encrypted and forgotten.
  So anyone with **runtime** access to the running Service (container
  shell, a code-execution vulnerability, SSH to the host) has the same
  decryption capability the app itself has; at-rest encryption only
  protects against *partial* compromise — someone getting just the
  SQLite file (a leaked backup, a copied dev database) without also
  getting the encryption keys, or purely accidental exposure. A full
  host compromise defeats it, same as it would for any single-box
  self-hosted app without a separate KMS/HSM — not solvable here without
  contradicting the "deployable anywhere, no cloud dependency, trivially
  self-hostable" goals.
- **Decision: Active Record encryption keys are sourced from Kamal
  secrets (env vars), not `config/credentials.yml.enc`.** Rails supports
  configuring `active_record.encryption.primary_key` /
  `deterministic_key` / `key_derivation_salt` directly from `ENV`
  instead of the credentials file. The actual key values then live in
  `.kamal/secrets` (already part of this repo's deploy setup, pulling
  from whatever local secret store the deployer uses) rather than a
  file shipped inside the application code. Doesn't stop a live
  container compromise — the running process still needs the key in
  memory to function, true of any encryption scheme — but it does mean
  a `git clone`, a repo access leak, or a "back up the whole `service/`
  directory" mistake no longer carries the keys alongside the encrypted
  data. Considered and rejected: `dotenvx`-style committed-but-encrypted
  env files — that tool solves "safely commit secrets to git," which
  isn't a problem this design has; Kamal secrets already keep key
  material out of the repo entirely, a stronger position than encrypted-
  but-still-committed.
- **Decision: an admin "revoke calendar credentials" action** (per-room
  or all-at-once, for a suspected full compromise) — calls the
  provider's token revocation endpoint (Google: direct
  `oauth2.googleapis.com/revoke` call; Microsoft: coarser, effectively
  pulling the app's consent grant), clears the room's `config`, bumps
  `credential_version` so the App stops using its cached copy, and
  records a `calendar_credential_revoked` `LogEntry`. This is the actual
  backstop once the threat model above is accepted: no amount of
  at-rest encryption prevents a fully-compromised host from reading a
  secret the app itself must be able to use — revoking at the provider
  is what makes a stolen token stop being useful, the same role
  per-device API token revocation already plays for a lost/stolen
  Apple TV (see `Device` notes), just scoped to the calendar credential
  specifically and covering the larger blast radius of a Service-side
  compromise (every room at once) rather than one physical device.

### Emergency notifications (building safety systems)

- **Positioning: this is a supplementary/nice-to-have channel, not the
  primary emergency notification endpoint.** The space the Apple TV is in
  is assumed to already have adequate primary safety notification
  infrastructure independent of this App. This recontextualizes every
  decision below: gaps and accepted tradeoffs (push-only delivery with no
  reconciliation, the AirPlay-can-cover-an-active-alert edge case) are
  reasonable specifically *because* nobody's safety depends solely on this
  screen.
- **Requirement**: the App must interface with building safety/mass
  notification systems (Singlewire InformaCast and comparable competing
  products) and display emergency notifications (lockdown, etc.) on
  screen.
- **Pluggable, not Singlewire-specific**: designed as a generic
  `EmergencyNotificationProvider` interface on the Service, mirroring
  `CalendarProvider`. Known competing products in this space: Singlewire
  InformaCast, Rave Mobile Safety, Everbridge, Omnilert, Regroup, Centegix
  CrisisAlert. Only InformaCast's integration mechanics have actually been
  verified so far; the others are assumed similar (API/webhook-based) but
  not yet confirmed individually.
- **InformaCast integration (verified)**: supports both push (outbound
  REST/webhook call, "Universal API Connector") and pull (RSS/CAP feed)
  models for third-party displays/signage.
- **This is categorically different from calendar sync in criticality**:
  latency tolerance is seconds, not minutes, and a missed alert is a
  safety failure, not a stale-data inconvenience.
- **Scope is per-alert, not per-integration, and targeting is the
  vendor's responsibility, not ours.** The vendor connection itself
  (credentials, webhook URL) is unscoped — whoever sends a given
  broadcast picks its target(s) at send time (the whole organization, one
  or more sites, buildings, or individual rooms, in any combination),
  using our exposed `Organization`/`Site`/`Building`/`Room` hierarchy as
  their recipient groups/zones. We don't model targeting relationally
  ourselves.
- **Architecture**: the Service holds the vendor integration (receives
  webhooks, fans out to every paired App) — the App (a TV behind NAT)
  can't receive inbound webhooks directly.
- **Decision: the App also gets a direct fallback path to the provider**,
  independent of the Service, so an alert can still reach the screen
  during a Service outage. **Important limitation**: this only works for
  providers that expose a *pull*/polling endpoint the App can hit directly
  (confirmed for InformaCast via its RSS/CAP feed). A webhook-only vendor
  gives the App nothing to poll, so it stays Service-dependent regardless
  of how the App is built.
- **Decision: delivery is APNs-only, no polling fallback.** Uses a proper
  high-priority **alert-type** APNs push (not the low-priority
  `content-available` silent push used for calendar nudges, which the OS
  can throttle). Considered and rejected adding emergency status to the
  `rooms/current` poll as a reconciliation floor: APNs is technically
  best-effort, not guaranteed, so a poll fallback would close a real (if
  rare) gap — but a second delivery path means reconciliation logic and
  race conditions to get right, and for safety-critical software, an
  unverified complex system can be a worse bet than a simple,
  well-understood one. **Accepted tradeoff**: a dropped push isn't caught
  until some other state change happens.
- **Display treatment when an alert is active**:
  - **Full-screen takeover**, not an overlay — high-contrast alert
    styling, no calendar content competing for attention.
  - **Not dismissible via the Siri Remote** — only a server-driven
    "cleared" push ends it.
  - **Push payloads carry an `alert_id`** to correlate a "triggered" push
    with its later "cleared" push. The App tracks a small local set of
    currently-active alert IDs and shows the takeover whenever that set is
    non-empty.
  - **Active-alert state is persisted locally on-device.** If the App
    restarts mid-emergency, it resumes showing the alert from local
    storage rather than silently reverting to the calendar view until
    some other push arrives.
  - Calendar polling keeps running normally in the background while an
    alert is showing.
  - **Known, accepted gap**: there's no public tvOS API for an app to
    dynamically block or preempt an incoming AirPlay session, so someone
    AirPlaying at the exact moment an alert is active could visually
    cover it. Accepted given this channel's supplementary positioning.

## API contract (the App ↔ the Service)

All authenticated endpoints use `Authorization: Bearer <api_key>`;
everything is under `/api/v1/`.

- **Pairing Code flow (public, unauthenticated)**:
  - `POST /devices/pairing_codes` → creates a pending `Device`, returns
    `{device_id, code, expires_at, poll_interval_seconds}`.
  - `GET /devices/:device_id/pairing_status` → `{status: "pending"}` until
    claimed, then `{status: "paired", api_key, room: {...}}`.
- **Zero-Touch flow (public, token-based)**:
  - `POST /devices/mdm_register` — body `{mdm_token, device_identifier}`
    (from Managed App Configuration) → `{api_key, room: {...}}`
    immediately, no polling.
- **Ongoing operation (authenticated)**:
  - `GET /rooms/current` — the main 1-minute baseline poll. Returns room
    name/timezone/background image, and calendar source config (enough
    for the App to poll the provider directly, per Option A). Calendar
    credentials (refresh token) are only included when a
    `credential_version` the App sends back doesn't match current —
    avoids retransmitting a sensitive token on every poll once the App
    already has it. Doubles as the heartbeat (`last_seen_at`). **Does
    not** carry emergency alert data (APNs-only, see above).
  - `POST /devices/push_token` — registers/updates the App's APNs device
    token; called at launch and whenever the token rotates.
  - `POST /calendar_sources/sync_status` — the App reports the outcome of
    its own most recent calendar poll (per Option A, only the App can
    actually observe this) for its paired room's `CalendarSource`. Body:
    `{status: "ok"}` or `{status: "error", error: "..."}`. Always updates
    `CalendarSource.last_reported_at` (regardless of status) and
    `last_synced_at` on success; only creates a `LogEntry`
    when the reported status differs from the previous one (see
    `LogEntry`'s `calendar_sync` notes below).

## Data model

No `account_id`/tenancy field on anything below. No separate
`PairingCode` model (see `Device` notes) and no `paired_via` field (see
`Device` notes).

### `Organization`

| Field | Type | Notes |
|---|---|---|
| `name` | string | |
| `support_information` | rich text (Action Text + Lexxy) | optional — markdown-style editing, stored as HTML. E.g. admin/tech support contact info for the deployment |

Top of the physical hierarchy. Always exactly one row per Service
instance — this is not a multi-tenancy mechanism, it's naming the single
tenant explicitly rather than leaving it implicit (see Product shape/
goals: no multi-tenancy at the Service level, ever). `has_many :sites`.

### `Site` (`belongs_to :organization`)

| Field | Type | Notes |
|---|---|---|
| `name` | string | a collection of buildings |

Unique on `organization_id` + `name` — same normalization rationale as
`Floor` (see below): buildings need to reference the *same* site
consistently, and a duplicate-but-distinct site record with the same name
would be a real ambiguity, not just cosmetic.

### `Building` (`belongs_to :site`)

| Field | Type | Notes |
|---|---|---|
| `name` | string | |
| `timezone` | string | IANA identifier (e.g. `America/Chicago`) — lives here, not on `Room`: all rooms in one physical building share a timezone, so per-room would risk two rooms in the same building drifting to different zones |

Unique on `site_id` + `name` — same normalization rationale as `Floor`
(see below): floors need to reference the *same* building consistently.

### `Floor` (`belongs_to :building`)

| Field | Type | Notes |
|---|---|---|
| `name` | string, required | e.g. "1", "Ground", "Mezzanine", "B1" — every floor has a name, including a single-story building's only floor (no null/unnamed-floor concept) |
| `position` | integer, required | display sort order, managed by the `positioning` gem scoped to `building` — DB-enforced unique per building via a `building_id` + `position` unique index, assigned automatically (next available slot) on creation, resequenced automatically on reorder. Backs a drag-and-drop reordering UI, not a user-typed field |

Unique on `building_id` + `name`. Reversed from an earlier plain-string
`floor` field on `Room`: many rooms
in the same building need to reference the *same* floor consistently, and
free text gives no mechanism to prevent "3" / "Floor 3" / "3rd Floor" all
describing the same physical floor — a real normalization case. Admins
pick from the building's existing floors rather than retyping one each
time.

### `Room` (`belongs_to :floor`)

| Field | Type | Notes |
|---|---|---|
| `name` | string | |
| `background_image` | ActiveStorage attachment | |

Every room belongs to a real `Floor` record — a single-story building
still creates one real `Floor` (e.g. named "1") rather than a null
relationship, so there's no null-relationship case to special-case in
queries (e.g. "group rooms by floor"). No separate `building_id` on
`Room`: a building is always reachable via `room.floor.building`, so
storing it redundantly would just recreate the two-sources-of-truth
problem this avoids — `floor_id` alone is the single source of truth for
which building a room is in.

Unique on `floor_id` + `name` — beyond the same normalization concern as
`Site`/`Building`/`Floor`, a duplicate room name on the same floor would
be directly user-facing: the Apple TV display exists specifically to
answer "is *this* room free," and two same-named rooms on one floor make
that ambiguous.

**UI note — progressive disambiguation when displaying a room's
identity** (admin UI room lists/pickers; not the App's own self-display,
which always shows just its one paired room): show only as much of the
`Site → Building → Floor → Room` hierarchy as is actually needed to
disambiguate given the current data shape, not always the full path.
E.g. "Everest" alone is enough when there's only one `Site` with only one
`Building` with only one `Floor`, since no duplicate is structurally
possible (per the `floor_id` + `name` uniqueness above); once a building
has multiple floors, prefix with the floor ("Floor 3 → Everest"); once a
site has multiple buildings, prefix with the building too ("Bldg A →
Floor 3 → Everest"); once the organization has multiple sites, prefix
with the site too ("HQ → Bldg A → Floor 3 → Everest"). The rule is fully
recursive all the way to `Site` — a level's prefix is added only once
that level's own collision becomes structurally possible.

The Service supports exporting and re-importing the full
`Organization`/`Site`/`Building`/`Floor`/`Room` hierarchy from the admin
UI (a versioned JSON schema, not just a human-readable dump — it must
stay machine-parseable long after export, possibly after the data model
has changed). Exports include
`CalendarSource.provider` + `calendar_id` for Google/Microsoft sources and
the full URL for ICS sources, but never OAuth refresh tokens. Device
pairing never round-trips through the file either way — a physical Apple
TV has to actually re-authenticate regardless of import.

### `CalendarSource` (`belongs_to :room`)

| Field | Type | Notes |
|---|---|---|
| `provider` | enum | `google` / `microsoft` / `ics` |
| `config` | encrypted (Rails 8 `encrypts`) | refresh token + calendar ID for Google/Microsoft; just the URL for ICS |
| `credential_version` | integer | bumped on change; matches the `rooms/current` field the App compares to know when to refetch |
| `webhook_subscription_id` | string, nullable | Google/Microsoft webhook subscriptions expire and need periodic renewal by a background job |
| `webhook_expires_at` | datetime, nullable | see above |
| `last_reported_at` | datetime, nullable | updated on *every* report to `POST /calendar_sources/sync_status`, success or failure — the true heartbeat, mirroring `Device.last_seen_at`. When the current status (from the most recent `calendar_sync` `LogEntry`) is `ok`, this doubles as the exact last-successful-sync time, since the most recent report must have been the success that produced that status. No separate `last_synced_at` column — it would just be another copy of information already recoverable from `last_reported_at` plus the transition log |

**Derived staleness, not stored.** `stale?`
(`last_reported_at.nil? || last_reported_at < 1.hour.ago`) is computed at
render time on the Service side, never persisted — same reasoning as
`last_sync_status`: fully derivable, so a stored column would just be
another copy of the truth to keep in sync. Admin UI status display is
three-way, not a binary red/green: `stale?` true → gray/amber "unknown"
regardless of the last logged status (a confident green is misleading if
we haven't heard from the room in over an hour); otherwise, green/red
from the most recent `calendar_sync` `LogEntry`'s status. One hour is a
monitoring threshold, tuned for admins to find out same-day without
flapping on a transient blip — see the App's own on-device staleness
indicator (Calendar syncing, above) for the separate, much more
conservative threshold used for what's shown to someone standing in the
room.

Kept as its own model rather than folded into `Room` as columns
(reversing an earlier draft of this decision): `Organization`/`Site`/
`Building`/`Floor`/`Room` are structural — they exist so admins can
organize and navigate to the right physical room — but calendar data is
the actual functional payload the whole system runs on, and deserves to
be modeled with real constraints rather than as loosely-typed hierarchy
scaffolding. Concretely: keeping it separate lets `provider` and `config`
be genuine `NOT NULL` columns — the row's existence *is* the "this room
has a calendar configured" check, with no way to represent a
half-configured state (a `provider` with no `config`) at the schema
level. Folded into `Room` as nullable columns, that pairing could only be
enforced via an app-level conditional validation, not a real schema
guarantee. Persisted on the Service even though the App does the ongoing
polling per Option A: the Service needs it for the admin UI, for handing
to a replacement device if a TV is swapped, and because the OAuth
consent flow itself has to happen through a browser (the Service), not
the TV.

### `Device`

| Field | Type | Notes |
|---|---|---|
| `device_identifier` | UUID | App-generated on first launch |
| `room_id` | FK, nullable | null until paired |
| `status` | enum | `pending` / `paired` / `revoked` |
| `api_key` | string | stored hashed/digested, not plaintext — compared on every request but unrecoverable from a DB leak, same principle as password storage |
| `apns_token` | string, nullable | |
| `mdm_device_id` | string, nullable | non-null ⟺ paired via MDM — see "no `paired_via`" below |
| `pairing_code` | string, nullable | |
| `pairing_code_expires_at` | datetime, nullable | |
| `paired_at` | datetime, nullable | covers both pairing paths (code or MDM) with one field |
| `last_seen_at` | datetime, nullable | |

- **No separate `PairingCode` model** — folded into `Device` as columns.
  A device only ever has one active code at a time, and `paired_at`
  already covers both pairing paths with a single field, so a dedicated
  table would only buy an audit trail of past pairing attempts, which
  isn't valuable here.
- **No `paired_via` field** — redundant with `mdm_device_id` presence:
  non-null means MDM, null means pairing code. Derive it from
  `mdm_device_id.present?` wherever needed.
- **Pairing code expiry**: codes do expire (moderate window, ~15–30 min)
  — mainly to bound the brute-force window on a short code and avoid
  abandoned pending devices accumulating forever, not because the code
  alone is exploitable (claiming a device still requires valid OIDC admin
  auth). The App auto-requests a fresh code shortly before the current
  one expires while still on the pairing screen, so whoever's setting it
  up never sees an expiry error.

### `LogEntry`

| Field | Type | Notes |
|---|---|---|
| `event` | string/enum | `emergency_alert`, `calendar_sync` for now; extensible to other kinds of service-wide log entries later without a schema change |
| `occurred_at` | datetime | |
| `message` | text, nullable | for `emergency_alert`: delivered directly in the APNs alert payload (4KB limit is plenty). For `calendar_sync`: a human-readable summary, e.g. "Calendar sync recovered" or the reported error text |
| `payload` | JSON, nullable | raw source data — for `emergency_alert`, the incoming webhook body as received |
| `loggable` | polymorphic, nullable (`loggable_type` + `loggable_id`) | the subject of the event, when it has exactly one — e.g. the `CalendarSource` for a `calendar_sync` entry. Null for event types with no single subject (`emergency_alert` fans out to many rooms; targeting is resolved from `payload` instead, see below). Same pattern as ActiveStorage's own `record_type`/`record_id` columns already in this schema. No DB-level foreign key constraint is possible on a polymorphic column, unlike every other association in this schema — accepted here because `LogEntry` is an append-only historical record, not a live relationship anything else depends on |

A simple append-only log, not stateful — no `resolved_at`/ongoing concept.
Consistent with there being no server-side "current status" to query for
emergency alerts (APNs-only delivery, no reconciliation polling): a
"triggered" and a later "cleared" are just two separate log entries, not
one row with changing state.

**No dedicated `EmergencyAlert` model** — it's one `event` value of a
general-purpose `LogEntry`, not a bespoke table.

**`calendar_sync` entries are transition-only, not logged on every
report.** The App calls `POST /calendar_sources/sync_status` roughly
once a minute (matching its own calendar poll baseline) — logging every
call would mean thousands of near-identical "still ok" rows per room per
day, working against the Service's own "trivially self-hostable, single
SQLite file" goal, and burying the one transition an admin actually
wants to see under noise. A new `LogEntry` (`loggable: calendar_source`)
is only created when the reported status differs from
`CalendarSource.last_sync_status` (derived from the most recent
`calendar_sync` `LogEntry` for that `CalendarSource` — no separate status
column on `CalendarSource` itself, to avoid the same
two-sources-of-truth problem avoided elsewhere in this schema).
`CalendarSource.last_reported_at`, by contrast, is updated on every
report (success or failure), not just transitions — status answers "is
it currently broken," `last_reported_at` (which doubles as
last-successful-sync time when status is `ok`, per the `CalendarSource`
section above) answers "how fresh is this," and transition-only logging
can only give you the first one (a long unbroken streak of successful
pings, or a device that's gone silent entirely, would look identical in
the log otherwise).

**No `EmergencyAlertTarget` model** — targeting is the vendor's
responsibility, not ours (see Emergency Notifications above). When a
vendor's webhook fires, the payload carries which of our IDs it's
targeting; we resolve that directly against the `LogEntry`'s `payload` at
delivery time for fan-out, with nothing persisted as a relational
structure of our own. Fan-out resolves every target to its room set (an
`Organization` target expands to every room in the deployment — there's
only ever one `Organization` row per Service instance, so this is
equivalent to "everyone"; a `Site` target expands to all rooms in all its
buildings, a `Building` target to all its rooms, a `Room` target to
itself), unions those sets so a room covered by more than one target is
only pushed once, then sends the APNs push to each resulting room's
paired `Device`.

## Open questions

- Emergency notification provider implementations beyond InformaCast
  (Rave, Everbridge, Omnilert, Regroup, Centegix — integration mechanics
  not yet individually verified), including the vendor-connection/
  credentials model itself (an `EmergencyNotificationSource`-type model —
  not yet designed even for InformaCast; `LogEntry` only covers recording
  the alert event, not how the Service authenticates to the vendor).

## Repository layout

Each main component is its own git repository — not a monorepo — but all
three live as nested directories under one local parent folder
(`whatsupnext/`) so tooling that can't navigate above its working
directory (e.g. the Claude Code CLI) can still reach all of them:

```
whatsupnext/       — umbrella repo: README.md + SPEC.md only, no code.
                      Describes the Service and the App together
                      (including the API contract between them), since
                      that shared context doesn't belong to either repo
                      individually.
  service/          — the Service (Rails), own repo
  app/              — the App (tvOS, Xcode project), own repo
  saas/             — the SaaS control plane, own repo, closed-source
```

`service/`, `app/`, and `saas/` are each independent git repos (their own
`.git`) and are excluded via the umbrella repo's `.gitignore` — nesting
them locally for tooling convenience does not merge them into the
umbrella repo or into each other. On GitHub, the equivalent grouping is a
GitHub Organization: independent repos under one shared namespace, each
with its own visibility (`service`/`app` public, `saas` private).
