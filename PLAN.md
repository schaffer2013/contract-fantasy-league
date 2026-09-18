# Contract Fantasy League — Implementation Plan

## 1. Scope and boundaries

This plan covers **Project L**, the live fantasy football product used by real commissioners and owners.

Project L will have its own:

- Git repository and release history
- Application runtime and deployment
- PostgreSQL database and migrations
- Background workers and job queues
- secrets, provider credentials, monitoring, and backups
- operational event and audit history

The historical simulator, **Project H**, remains a separate research application. A future shared package may contain stable enums, schemas, player-ID mapping helpers, classifier logic, and test fixtures. Project L must never query the Project H database or require the simulator to operate.

### First-release outcome

The first usable release should let a commissioner create and configure a league, assign 12 franchises, run a startup auction during any part of the season, activate the league on the correct future week, and operate lineups, scoring, standings, waivers, trades, money, and keeper contracts through a season. Owners and the commissioner will initially enter through a development-only identity switcher rather than real authentication.

### Explicit early non-goals

- Public multi-tenant signup and billing
- Native mobile applications
- Gambling, paid contests, or prize custody
- Historical-simulation execution inside the live product
- Fully automated judgment of ambiguous NFL contract events
- Permanent production use without authentication

## 2. Settled product rules

### League structure

- Default league size: 12 franchises
- Default annual cap: $200 per franchise
- Roster limit: 20 players
- Starting lineup: 1 QB, 2 RB, 2 WR, 1 TE, 1 FLEX, and 1 K
- FLEX eligibility: RB/WR/TE by default
- Bench: 12 players under the default lineup
- Team defense: disabled
- Commissioner may edit roster and scoring settings

### Baseline scoring template

The configuration UI will expose every value. The initial template is:

| Category | Default points |
| --- | ---: |
| Passing yards | 1 per 25 yards |
| Passing touchdown | 4 |
| Interception thrown | -1 |
| Rushing yards | 1 per 10 yards |
| Rushing touchdown | 6 |
| Reception | 1 |
| Receiving yards | 1 per 10 yards |
| Receiving touchdown | 6 |
| Fumble lost | -2 |
| Extra point made | 1 |
| Field goal, 0–39 yards | 3 |
| Field goal, 40–49 yards | 4 |
| Field goal, 50+ yards | 5 |

Fractional and negative scoring support should be part of the scoring engine even when a template uses whole numbers.

### Weekly competition

- Every team has one scheduled head-to-head opponent per scoring week.
- When beat-the-median is enabled, every team also receives a second weekly result against the median of all franchise scores.
- The median calculation and tie behavior must be deterministic and shown in league settings.
- Standings retain head-to-head and median results separately as well as in the combined record.
- Schedule and standings calculations must support a shortened season for leagues created midyear.

### Money and contracts

- The annual auction budget and current-season waiver money use a single auditable money model.
- At auction close, every unspent auction dollar becomes that franchise's waiver money.
- Auction purchases create keeper-eligible fantasy contracts at the winning salary.
- In-season waiver and free-agent acquisitions are rentals and are auction-bound for the next annual auction.
- Current-season money may be included in trades.
- Trade validation must evaluate both teams atomically after players and money move.
- Current-season transferred money does not roll into the next league year.
- No team may spend or trade money it does not have.

### Transaction deadline

- The default transaction deadline is exactly four fantasy weeks before the configured playoff start.
- The commissioner can configure the playoff start and review the calculated deadline.
- The transaction types affected by the deadline will be explicit settings; the initial assumption is that trades close at the deadline while lineup moves and normal injury replacements remain available under their own rules.

### Midseason startup

The system will derive a proposed activation week when the startup auction closes:

1. Read the auction completion time in the configured league timezone.
2. If completion is on or before the Wednesday cutoff, select the next NFL scoring week.
3. If completion is after the Wednesday cutoff, skip the immediately upcoming week and select the following NFL scoring week.
4. Never award retroactive points for weeks before activation.
5. Show the proposed activation week to the commissioner and require confirmation.
6. Store the calculation inputs, selected week, confirming actor, and timestamp in the audit ledger.

The exact Wednesday time is a setting. The initial default will be Wednesday at 11:59 p.m. in the league timezone until the commissioner changes it.

## 3. User experiences

### Owner dashboard

The landing page should answer: Who do I play, is my lineup legal, how much money do I have, and what needs attention?

- Current and next matchup
- Live/official team score and league median progress
- Lineup completeness and player game locks
- Available waiver money and active commitments
- Pending trades, waiver results, and contract alerts
- Recent league activity

### Team and lineup management

- Drag/drop and accessible form-based lineup changes
- Position and FLEX eligibility validation
- Per-player game locks based on actual kickoff
- Roster limit, duplicate-player, and salary legality checks
- Player availability, injury status, NFL team, bye week, fantasy salary, keeper state, and expected NFL contract end
- Clear reasons when the server rejects a move

### Player market and details

- Search and filters by name, position, NFL team, ownership, salary, status, and availability
- Player detail with stats, game log, news/provider timestamps, fantasy transaction history, and NFL contract-event history
- One reusable contract-status presentation across roster, search, auction, waivers, trades, history, and admin tools

### Live auction room

- Commissioner starts, pauses, resumes, and closes the auction
- Round-robin nomination with configurable nomination and bid timers
- Server-authoritative bids delivered in real time
- Hard cap and minimum-roster-fill constraints on every bid
- Reconnection restores canonical state without duplicating bids
- Commissioner correction workflow records reversals and replacement results
- Auction completion converts remaining dollars into waiver money and calculates league activation

### Waivers and free agency

- Blind bids with active commitments visible only to the submitting team
- A team cannot commit more than its available waiver money
- Configurable weekly processing schedule and deterministic tiebreaker
- Winning amount becomes the rental player's current-season salary
- Losing commitments are released atomically
- Cleared players may enter configurable free agency until their individual game lock
- Release timing cannot retroactively fund an already-open waiver run

### Trades

- Players and current-season money in either direction
- Contract salary and keeper/rental status transfer unchanged with the player
- Server preview of both teams' post-trade cap, roster, and lineup legality
- Offer, counter, reject, cancel, accept, commissioner review if enabled, and expiration states
- Atomic processing and complete trade history

### Commissioner portal

- League setup wizard and settings version history
- Franchise creation, owner labels, colors, and debug-access mapping
- Schedule generation and manual correction
- Auction controls and correction tools
- Scoring, stat corrections, and matchup finalization
- Roster, cap, waiver, trade, and lineup overrides with mandatory reasons
- NFL contract-event inbox, raw source view, normalized interpretation, and override actions
- Provider freshness, failed jobs, unmatched players, reconciliation conflicts, and alerts
- Searchable audit log with before/after values and actor identity

## 4. Proposed architecture

### Repository layout

```text
contract-fantasy-league/
  apps/
    web/                 # Next.js UI and HTTP/realtime entry points
    worker/              # scheduled jobs, ingestion, scoring, reconciliation
  packages/
    domain/              # rules, state machines, money, contracts, scoring
    database/            # schema, migrations, repositories, projections
    providers/           # NFL data/stat/notification adapters
    ui/                  # shared components and design tokens
    config/              # typed environment and league defaults
    test-fixtures/       # deterministic provider and league scenarios
  docs/
  infra/
  docker-compose.yml
```

### Runtime components

1. **Web application** — renders owner/admin pages, handles commands and queries, and maintains live auction/score connections.
2. **Domain services** — enforce roster, money, scoring, keeper, waiver, trade, and permission rules independent of UI and vendors.
3. **PostgreSQL** — canonical events, current projections, league configuration versions, jobs, and audit data.
4. **Redis/job queue** — scheduled work, retries, distributed locks, auction timers, waiver runs, and short-lived fan-out state.
5. **Worker** — provider polling, normalization, reconciliation, scoring, finalization, notifications, and projection rebuilds.
6. **Provider adapters** — isolate vendor payloads and identifiers from the domain model.

### Consistency approach

- Commands run in database transactions and append immutable domain/audit events.
- Current roster, cap, standings, and contract projections are derived read models optimized for the UI.
- Critical workflows use idempotency keys and database uniqueness constraints.
- Auction bids, waiver resolution, trades, and commissioner corrections use server-side locks or serializable transactions.
- External events first enter an immutable raw store and are normalized asynchronously.
- The site remains usable when an external NFL provider is unavailable.

## 5. Core domain model

Initial entities and ledgers should include:

- `League`, `LeagueSeason`, `LeagueSettingsVersion`, `FantasyWeek`
- `Franchise`, `OwnerIdentity`, `FranchiseSeason`
- `Player`, `NFLTeam`, `PlayerExternalId`
- `RosterAssignment`, `LineupSlot`, `LineupSubmission`
- `FantasyContract`, `KeeperProjection`
- `MoneyLedgerEntry`, `MoneyBalanceProjection`
- `Auction`, `AuctionNomination`, `AuctionBid`, `AuctionSale`
- `WaiverRun`, `WaiverBid`, `WaiverAward`, `FantasyRelease`
- `Trade`, `TradeAsset`, `TradeDecision`
- `Matchup`, `PlayerScore`, `TeamScore`, `MedianResult`, `Standing`
- `RawProviderEvent`, `NFLContractEvent`, `NFLContractState`
- `CommissionerReview`, `ManualOverride`, `AuditEvent`
- `ScheduledJob`, `ProviderCursor`, `ReconciliationRun`

Use explicit namespaces in code and storage. A fantasy trade is never stored as an NFL transaction, and an NFL trade is never stored as a fantasy trade.

## 6. NFL data and contract-event pipeline

### Ingestion flow

```text
Provider poll or snapshot
  -> immutable raw payload
  -> normalize and map player IDs
  -> deduplicate canonical NFL event
  -> classify qualifying/non-qualifying/review-required
  -> update current NFL contract projection
  -> update affected fantasy keeper projection
  -> notify UI/admin and append audit event
```

### Operational baseline

- Contract-event polling interval: configurable, initially 15 minutes
- Target detection: within 60 minutes of provider publication
- Full current-state reconciliation: at least daily
- Retry with bounded exponential backoff
- Provider cursor and last-success visibility in the admin portal
- Manual queue for ambiguous classifications, mapping failures, and conflicting snapshots

### Contract classification baseline

Normally qualifying:

- New NFL contract
- Same-team extension
- Material renegotiation of years, gross value, pay, or guarantees
- NFL trade
- Release/termination followed by a new contractual relationship
- Commissioner-approved substantive event

Normally non-qualifying:

- Pure cap-accounting restructure with no substantive economic change

Unknown or conflicting events become `REVIEW_REQUIRED`. Overrides append actor, timestamp, prior value, new value, source, and reason; they do not delete the original event.

## 7. Access, authorization, and backdoor retirement

### Development phase

- `DebugIdentityProvider` presents a team/admin switcher.
- Every request still carries an explicit identity, role, league, and franchise scope.
- Domain authorization checks apply as if authentication were present.
- Debug entry and identity switches are audited.
- Debug mode is enabled only through a development/test environment setting.
- Production startup fails closed if debug access is enabled.

### Authentication phase

Replace only the identity provider and session transport. Add invitations, account linking, commissioner and owner roles, recovery, session revocation, and optional social/email login without changing domain authorization rules.

Before any public deployment, complete a threat model covering cross-league access, privilege escalation, forged bids, race conditions, replayed commands, provider webhook validation, secret handling, and audit integrity.

## 8. Delivery phases

### Phase 0 — Foundation and executable rules

- Initialize the monorepo, formatting, linting, type checking, tests, and CI
- Add local PostgreSQL and Redis development services
- Define typed league settings and the default 12-team template
- Model league weeks, shortened seasons, and the Wednesday activation calculation
- Implement append-only audit and money ledgers
- Add deterministic tests for roster, cap, scoring, median, and activation rules

**Exit criteria:** A test suite proves the settled league rules without a UI or external data provider.

### Phase 1 — League, franchise, and debug portals

- Commissioner league-creation wizard
- Franchise setup and 12-team default
- Debug identity switcher for commissioner and each owner
- Owner shell, commissioner shell, navigation, and responsive design system
- Settings editor with versioning and validation
- Player catalog using fixture data behind the provider interface

**Exit criteria:** A commissioner can create a configured league and enter every portal locally without authentication.

### Phase 2 — Startup auction and midseason activation

- Auction state machine, nomination rotation, timers, bidding, and hard-cap validation
- Real-time room and reconnect behavior
- Roster-fill affordability constraints
- Commissioner pause, correction, and close controls
- Remaining auction dollars converted to waiver money
- Midseason activation proposal and commissioner confirmation

**Exit criteria:** Twelve browser sessions can complete a legal auction, recover from reconnects, and activate on the expected week.

### Phase 3 — Lineups, scoring, matchups, and standings

- Schedule generator that supports full and shortened seasons
- Lineup editing and per-player kickoff locks
- Stats adapter, scoring engine, provisional/live/final states, and corrections
- Optional beat-the-median result
- Standings and playoff qualification projections
- Four-week-before-playoffs transaction-deadline calculation

**Exit criteria:** A complete week can progress from editable lineups through official scores, two-result standings, and audited corrections.

### Phase 4 — Waivers, free agency, releases, and trades

- Blind funded waiver commitments and scheduled resolution
- Release cutoff behavior and cleared free agency
- Rental contract creation
- Player/current-year-money trade workflow
- Atomic post-trade roster and balance validation
- Commissioner tools and transaction history

**Exit criteria:** No concurrency or retry scenario can overspend funds, duplicate ownership, or create an in-season keeper.

### Phase 5 — NFL contract monitoring

- Raw provider store and current-provider adapter
- Canonical event schema, deduplication, and classifier
- Current NFL contract projection and expected expiration display
- Keeper reset projection and reusable player contract-status component
- Admin review queue, overrides, alerts, and daily reconciliation

**Exit criteria:** Replaying duplicate events is harmless, ambiguous events require review, and accepted qualifying events update the correct future auction without changing current ownership.

### Phase 6 — Authentication and production readiness

- Real accounts, invitations, sessions, and role assignment
- Disable and prohibit the debug identity provider in production
- Rate limits, security headers, CSRF protection, and sensitive-action confirmation
- Backup/restore rehearsal and migration rollback procedure
- Structured logs, metrics, traces, alerts, and runbooks
- Load, accessibility, browser, and failure-injection testing
- Staging environment and production release checklist

**Exit criteria:** The threat model and release checklist pass, backups restore successfully, and production cannot boot with the debug backdoor enabled.

## 9. Testing strategy

### Domain tests

- Cap and roster invariants under every action
- Auction bids preserve money needed for required open roster slots
- Auction remainder converts exactly once to waiver money
- Waiver commitments never exceed available money
- Trade money and players settle atomically
- Only auction purchases create keeper eligibility
- In-season acquisitions always create rentals
- Qualifying NFL events alter future-auction status but not current ownership
- Median results handle even team counts and ties deterministically
- Settings versions reproduce prior-week scoring
- Midseason activation follows the Wednesday rule across timezones and daylight-saving transitions

### Integration and replay tests

- Duplicate HTTP commands, jobs, provider payloads, and reconnects are idempotent
- Projection tables rebuild from canonical events
- Provider outage and recovery do not interrupt team access
- Stat corrections recalculate only affected results and standings
- Commissioner reversals preserve original history
- Concurrent final bids, waiver runs, and trade accepts serialize safely

### Browser and accessibility tests

- Commissioner creates a league and configures rules
- Twelve owners complete an auction
- Owner sets a legal lineup and receives clear invalid-action feedback
- Owner submits/cancels waiver bids and completes a trade with money
- Commissioner reviews an ambiguous contract event
- Keyboard-only and screen-reader flows cover the auction, lineup, and waiver interfaces

## 10. Operations and observability

Track at minimum:

- Web and worker health, error rate, latency, and queue age
- Auction connection count, command rejection reasons, and timer drift
- Provider poll freshness and detection latency
- Reconciliation age and discrepancy count
- Failed player-ID mappings and review queue size
- Scoring completeness and stat-correction counts
- Projection lag and idempotency conflicts

Alerts should target actionable conditions: stuck auctions, overdue waiver runs, stale scoring, provider failures beyond threshold, unresolved high-confidence player mappings, and failed daily reconciliation.

Backups, point-in-time recovery, data-retention policy, migration rehearsals, and restoration drills are release requirements rather than post-launch cleanup.

## 11. Decisions to validate during implementation

These details should remain configurable or be confirmed through early prototypes:

- Playoff team count, playoff length, consolation format, and tiebreak order
- Exact Wednesday cutoff time and default league timezone
- Lineup behavior for postponed, suspended, or rescheduled NFL games
- Kicker missed-field-goal penalties and fractional distance scoring
- Injury reserve slots and eligibility rules
- Waiver schedule, tiebreak ordering, and minimum bid
- Trade review/veto policy and which transaction types close at the deadline
- Auction timer lengths, anti-sniping behavior, and nomination opening bid
- Source providers for live statistics, injuries, projections, player identity, and NFL contracts
- Notification channels and preferences

The system should not hard-code unresolved policy. Defaults belong in a versioned league template, with the commissioner shown the consequences before a season begins.

## 12. Definition of done for the initial live platform

The initial platform is complete when:

1. Project L deploys and operates independently of Project H.
2. A commissioner can configure the league and all 12 franchises.
3. Owners can enter their scoped portal through the temporary development access mechanism.
4. A server-authoritative startup auction works before or during the NFL season.
5. Midseason leagues activate on the configured future full week with no retroactive scoring.
6. Rosters, lineups, scoring, scheduled matchups, optional median results, standings, and playoffs work end to end.
7. Auction dollars, waiver money, salaries, commitments, and traded current-year money reconcile to an immutable ledger.
8. Waivers, free agency, releases, and trades enforce roster, contract, timing, and money rules under concurrency.
9. NFL contract events are ingested, deduplicated, classified, reconciled, reviewed, and reflected in keeper projections.
10. Commissioner overrides and scoring corrections are attributable and reversible without deleting history.
11. Critical jobs, provider freshness, and system failures are observable and alertable.
12. Real authentication replaces the debug identity provider before public production access.
