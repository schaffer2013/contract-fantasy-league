# Contract Fantasy League

A live, commissioner-managed fantasy football platform built around auction salaries and real NFL contract events.

This repository is **Project L**, the production application used by commissioners and fantasy owners. It is intentionally separate from **Project H**, the historical simulation and research system. The projects may eventually share a small, versioned package containing contract-event schemas and classification rules, but they will not share databases, runtimes, deployments, or production state.

## Current status

Planning only. The repository currently contains the product definition and implementation plan; application code has not started.

See [PLAN.md](PLAN.md) for the proposed architecture, domain model, milestones, and acceptance criteria.

## Product goals

- Run a complete 12-team contract fantasy football league.
- Allow a league to hold its initial auction at any point in the NFL season.
- Give each owner a focused team portal for lineup, roster, cap, waiver, trade, and contract management.
- Give commissioners an admin portal for rules, schedules, scoring, disputed events, overrides, and league-critical decisions.
- Track real NFL contract events and automatically project their effect on fantasy keeper eligibility.
- Preserve an immutable, understandable audit trail for money, ownership, contract, scoring, and commissioner actions.
- Deliver a familiar fantasy-platform experience while making salary and contract status visible throughout the product.

## Initial league defaults

All league settings must be editable by the commissioner unless a rule is explicitly marked as a system invariant.

| Setting | Default |
| --- | --- |
| Franchises | 12 |
| Annual salary cap | $200 |
| Roster size | 20 players |
| Starting lineup | 1 QB, 2 RB, 2 WR, 1 TE, 1 FLEX, 1 K |
| Bench | 12 players |
| Team defense | Disabled |
| Scoring | Conventional full-PPR defaults, commissioner-editable |
| Weekly matchups | Head-to-head, with optional additional result against the league median |
| Transaction deadline | Four weeks before the fantasy playoffs |
| Auction remainder | Converts to the team's in-season waiver budget |
| Trades | May include players and current-season money |

The starting scoring template will use common values such as 1 point per reception, 1 point per 10 rushing or receiving yards, 1 point per 25 passing yards, 4 points per passing touchdown, and 6 points per rushing or receiving touchdown. Kicker scoring and all scoring values will be explicit settings rather than hidden constants.

## Contract and money model

- The annual auction is the normal way a player becomes keeper-eligible.
- A player's winning auction bid becomes the player's fantasy salary.
- The fantasy salary remains fixed while the associated keeper contract remains active.
- A fantasy trade transfers the existing salary and contract status unchanged.
- A qualifying real NFL contract event makes an active keeper auction-bound for the configured future annual auction, without removing the player from the current fantasy roster.
- A cap-only NFL restructure does not reset keeper eligibility by default.
- Players acquired through in-season waivers or free agency are rentals and return to the next annual auction.
- Unused auction dollars become waiver money after the auction.
- Current-season money may be traded, remains auditable, and never rolls into a future season by default.

Real NFL events and fantasy transactions will be stored in separate ledgers. An NFL extension can change a fantasy contract's projected status, but it is not itself a fantasy transaction.

## Drafting during the season

A new league may complete its startup auction at any point in the year. No earlier NFL weeks are scored retroactively.

The initial activation policy is:

- If the auction finishes on or before the league's Wednesday cutoff, scoring begins in the next NFL week.
- If the auction finishes after the Wednesday cutoff, the immediately upcoming week is skipped and scoring begins the following NFL week.
- The commissioner sees and confirms the calculated activation week before finalizing the auction.
- The league timezone and exact Wednesday cutoff time are configurable and recorded with the activation decision.

This policy applies to new-league activation. Annual preseason auctions use the normal configured league calendar.

## Product areas

### Team portal

- Weekly matchup, lineup, projections, and scoring
- Roster with salary, keeper status, NFL contract expectation, and future-auction status
- Cap and waiver-money ledger
- Player search and player details
- Auction room, nominations, and bidding
- Blind waiver bids and free-agent acquisition
- Trade builder, current-year money, review, and history
- Standings, schedules, league median results, and playoffs
- Team activity and notifications

### Commissioner portal

- League creation and editable rule configuration
- Franchise and temporary owner-access management
- Schedule, activation week, playoffs, and transaction deadlines
- Auction controls, pause/resume, corrections, and nomination order
- Roster, lineup, cap, waiver, and trade administration
- NFL contract-event review and classification overrides
- Scoring corrections and stat-provider reconciliation
- Full audit history and system health

## Temporary access model

The first development version will intentionally have no user authentication. A clearly marked debug access screen will allow a tester to enter any team portal or the commissioner portal.

This backdoor is a development bootstrap mechanism, not a production security model. It must be controlled by environment configuration, visually obvious, audited, and impossible to enable accidentally in a production deployment. Authorization boundaries will still be designed from the beginning so real authentication can replace the debug identity provider without rewriting the domain services.

## Proposed technical direction

- TypeScript monorepo
- Next.js web application for owner and commissioner experiences
- PostgreSQL as the durable source of truth
- A separate background worker for NFL data ingestion, reconciliation, scoring, and scheduled league jobs
- Redis-backed jobs and distributed locks for auctions, waivers, and scheduled processing
- Server-authoritative real-time auction and score updates
- Containerized local development and deployable web, worker, PostgreSQL, and Redis services
- Automated unit, integration, browser, migration, and event-replay tests

Vendor-specific NFL data, statistics, projections, notifications, and hosting concerns will remain behind adapter interfaces.

## Guiding principles

1. Money and roster legality are enforced by the server, never trusted to the browser.
2. Important state changes are append-only events with derived read models.
3. Replayed jobs and duplicate provider messages are idempotent.
4. Ambiguous NFL contract events go to commissioner review instead of being guessed.
5. Commissioner overrides append compensating audit events rather than erasing history.
6. League settings are versioned so historical scoring and decisions remain reproducible.
7. External provider outages must not take down ordinary league access.
8. Project L never depends on the Project H database or runtime.

## Documentation

- [PLAN.md](PLAN.md) — product architecture and phased implementation plan

## License

No license has been selected yet. All rights are reserved until a license is added.
