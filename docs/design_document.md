# Football Match Recorder & Live Ticker — Architecture Design Document

**Document Type:** System Architecture & Design Specification
**Version:** 1.2 (MVP)
**Status:** Draft for Review
**Last Updated:** September 2026

### Version History

| Version | Summary of Changes |
|---|---|
| 1.0 | Initial architecture: SQLite, FastAPI/React, per-team auth, Google Sheets archival, deterministic report generation |
| 1.1 | Added goalkeeper-specific events and statistics (`save`, `goal_conceded`); added post-match coach/manager notes; introduced AI-assisted (Gemini) report narrative generation with deterministic fallback |
| 1.2 | Replaced SQLite with managed PostgreSQL as the primary datastore; generalized the report narrative engine into a pluggable provider architecture (Gemini and/or self-hosted Ollama, interchangeable); formalized modularity/extensibility as an explicit architectural principle |

---

## 1. Overview & Goals

### 1.1 Problem Statement
Match events (goals, cards, substitutions, etc.) are not currently recorded in a structured way during matches. Producing a match report retroactively from memory is unreliable, and manual post-match administrative work is undesirable for team members. The organization operates five teams, each of which must be able to record its own matches independently, occasionally concurrently.

### 1.2 Product Vision
A lightweight football match recording system that converts a small number of in-match interactions into automatically generated match reports and season-long team/player statistics, while providing a public live match ticker.

### 1.3 Core Design Principles
- **Minimum effort, useful result.** Event recording requires 2–4 taps, no typing, and no mandatory fields.
- **Progressive detail.** Additional optional input improves report quality, but minimal input still produces a valid, usable report.
- **No inference or fabrication.** Generated reports and statistics reflect only recorded data; no information is invented.
- **Recording as a byproduct.** Analytics and reporting are derived automatically from the primary recording action, rather than requiring separate data entry.
- **Modularity and provider-agnosticism.** External or swappable dependencies — the database engine, the report narrative generator, the archival destination — are accessed through internal abstraction layers, not called directly from business logic. This allows any one of them to be replaced or extended without rewriting the rest of the system (see Section 3.1).

### 1.4 Objectives
- Sub-5-second entry time for common events, optimized for one-handed mobile use.
- A correction-friendly workflow: undo-last, and a defined post-match correction window for editing.
- A public live ticker accessible without authentication.
- Automatically generated match reports, combining structured event data with optional coach/manager notes via an AI-assisted narrative layer.
- Automatically derived team and player statistics across a season, including goalkeeper-specific metrics.
- Continuous synchronization of match data to Google Sheets for long-term, human-readable archival.

### 1.5 Non-Goals (MVP)
- Advanced analytics (expected goals, possession, passing networks, heatmaps).
- Multiple simultaneous recorders per match.
- Native mobile applications (the system is a responsive web application).
- Direct API integration with TeamPlus (MVP relies on a copy/export workflow).
- Periodic database reset (superseded by continuous Sheets synchronization — see Section 12).

---

## 2. Personas & User Stories

| Persona | Description | Key Needs |
|---|---|---|
| **Bench Operator** | A team member responsible for recording events during a live match | Fast, large-target entry; no typing required; tolerant of input errors via undo/edit |
| **Public Viewer** | Spectators following a match remotely | Live score and event timeline; no authentication required; cross-device support |
| **Team Administrator** | Configures match details prior to kickoff | Rapid roster/lineup selection; management of the team's player list |
| **Club Administrator** | Oversees credentials and configuration across all five teams | Ability to manage team access codes via an administrative interface |

**Representative User Stories:**
- As a bench operator, recording a goal should require selecting "Goal," selecting the scorer, and optionally skipping further detail, completing in under five seconds.
- As a bench operator, the most recently recorded event should be reversible via a single action.
- As a public viewer, the live score and event timeline should update automatically without requiring a page refresh.
- As a team administrator, the starting lineup and substitutes should be configured prior to kickoff so that in-match player selection is limited to active participants.
- As a club administrator, team access codes should be manageable through a dedicated interface.

---

## 3. System Architecture

### 3.1 Architectural Principles: Modularity & Extensibility

The system is designed as a layered architecture with clear boundaries between business logic and any dependency that is likely to change, be swapped, or need extension over time. This is treated as a first-class architectural concern, not an incidental byproduct, so that the system can accommodate unanticipated future requirements without broad rewrites.

**Layering:**
- **API layer** (FastAPI route handlers) — request/response handling, authentication, input validation only. Contains no business logic or direct database access.
- **Service layer** — business logic (event recording rules, on-pitch roster derivation, report assembly, statistics computation). Depends only on abstract interfaces, never on concrete infrastructure.
- **Repository layer** — data access, implemented via SQLAlchemy models and queries. This is the only layer aware of the specific database engine in use.
- **Provider layer** — abstractions for external or swappable integrations, each defined as an interface with one or more concrete implementations selected via configuration:
  - `NarrativeProvider` — generates report prose from structured facts (implementations: Gemini API, self-hosted Ollama, or a no-op that defers to the deterministic summary; see Section 8).
  - `ArchiveSyncProvider` — synchronizes finalized match data to an external archive (implementation: Google Sheets; extensible to other destinations without touching match-finalization logic; see Section 12).

**Why this matters for this project specifically:** several decisions made during design (database engine, report generation method) have already changed once during the design process itself, before any code was written. A layered, interface-driven structure means such changes are configuration and implementation-swap concerns, not architectural rewrites — and the same protection extends to decisions not yet anticipated.

**Practical consequence:** adding a new capability (e.g., a local Ollama-based narrative provider, or a second archival destination) means writing one new class that satisfies an existing interface and updating a configuration value — not modifying the API layer, the service layer, or any other provider.

### 3.2 High-Level Component Diagram

```
                    ┌─────────────────────┐
                    │   Public Viewers      │
                    │  (browser, no          │
                    │   authentication)      │
                    └──────────┬───────────┘
                               │ HTTPS + WebSocket
                               ▼
┌──────────────┐      ┌─────────────────────┐      ┌──────────────┐
│ Bench          │◄────►│   React Frontend     │◄────►│ Team Admin    │
│ Operator App   │      │  (single-page app,    │      │ Setup UI      │
│ (mobile web)   │      │   statically served)  │      │               │
└──────────────┘      └──────────┬───────────┘      └──────────────┘
                               │ REST + WebSocket
                               ▼
                    ┌─────────────────────┐
                    │   FastAPI Backend     │
                    │  - Authentication      │
                    │  - Match/Event CRUD    │
                    │  - WebSocket hub       │
                    │  - Report generator    │
                    │    (pluggable provider)│
                    │  - Statistics engine   │
                    │  - Sheets sync service │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  PostgreSQL (managed) │
                    │  e.g. Neon free tier   │
                    │  (persistence decoupled│
                    │   from app hosting)    │
                    └──────────┬───────────┘
                               │ on match lock
                               ▼
                    ┌─────────────────────┐
                    │   Google Sheets       │
                    │  (per-team readable    │
                    │   archive, synced      │
                    │   automatically)       │
                    └─────────────────────┘

        Report generation calls out (post-match only) to whichever
        NarrativeProvider is configured — Gemini API or a self-hosted
        Ollama instance reached via tunnel — with a deterministic
        summary as the always-available fallback (see Section 8).
```

### 3.3 Technology Stack and Rationale

| Layer | Selection | Rationale |
|---|---|---|
| Backend | Python, FastAPI | Async-native (required for WebSocket support), automatic request validation via Pydantic, low operational overhead appropriate to project scale |
| Frontend | React (Vite) | Component-based state management is well suited to live-updating scoreboards, event timelines, and multi-step interaction flows |
| Styling | Tailwind CSS | Enables consistent, mobile-first, large-target UI without extensive custom CSS |
| Database | PostgreSQL (managed, e.g. Neon free tier) | Decouples data persistence from the application server entirely — the backend can be stateless and freely restarted/redeployed on a free hosting tier without any risk to data, since storage lives in a separate managed service. Accessed exclusively through the repository layer (Section 3.1), so the specific provider can change without touching business logic |
| Real-time updates | WebSockets (native FastAPI support) | Required for live ticker updates without polling; one channel per match |
| Authentication | Per-team access code exchanged for a short-lived session token (JWT) | Matches the requirement for minimal authentication complexity — five static credentials, no individual user accounts |
| Archival | Google Sheets API, behind an `ArchiveSyncProvider` interface | Provides a continuously updated, human-readable archive without requiring a periodic export/reset cycle; interface allows additional or alternative archive destinations later without touching match-finalization logic |
| Report narrative generation | Pluggable `NarrativeProvider`: Google Gemini API and/or self-hosted Ollama | Produces natural-language match reports grounded in structured event data and optional coach notes; called post-match only, off the live-recording critical path; provider is swappable via configuration (see Section 8) |
| Hosting | Render (free tier, upgradeable) for the FastAPI + React app; Neon (or equivalent) for managed PostgreSQL | GitHub Pages is a static-file host and cannot execute a backend process or maintain a database. As of 2026, Fly.io no longer offers a free tier for new accounts (a credit card and paid usage are required from signup); Render's free web service tier remains genuinely free but spins down after periods of inactivity, with a brief cold-start delay on the next request — acceptable for this project's usage pattern, since the application holds no state itself once the database is external |

**Note on hosting constraints:** Because the system requires a live backend process (authentication, coordinating database writes, WebSocket broadcasting), GitHub Pages cannot host the application directly. Splitting the application server (stateless, restartable, Render free tier) from the database (persistent, managed, Neon free tier) is a deliberate choice: it removes the need to manage a persistent disk for application-hosted storage — a genuine limitation on most free hosting tiers — by ensuring the only component that needs durable storage is a service purpose-built for exactly that.

### 3.4 Concurrency Model
The system supports multiple simultaneous live matches across different teams:
- Each match is assigned a unique `match_id` and a corresponding WebSocket channel; clients receive updates only for the match being viewed.
- PostgreSQL's native support for concurrent transactions and connection pooling comfortably handles the expected write volume (a small number of events per minute across up to five concurrent matches), with substantial headroom beyond current scale if usage grows.
- Session tokens are scoped to a single team; write access to a match is restricted to the owning team's session.

---

## 4. Data Model

### 4.1 Entity Relationship Overview

```
Team ──< Player
Team ──< Match
Match ──< MatchPlayer >── Player   (roster participation per match)
Match ──< Event >── Player          (primary actor)
Event ──> Player (secondary, nullable — e.g., assist, substitute-in)
```

### 4.2 Schema (PostgreSQL)

The schema is defined and versioned via SQLAlchemy models with Alembic migrations, ensuring changes are tracked and reversible. Native PostgreSQL types are used throughout (`JSONB` for variable-depth attributes, `TIMESTAMPTZ` for all timestamps, `BIGSERIAL` for primary keys to leave ample headroom for growth):

```sql
CREATE TABLE team (
    id                BIGSERIAL PRIMARY KEY,
    name              TEXT NOT NULL UNIQUE,
    access_code_hash  TEXT NOT NULL   -- bcrypt hash of the team's access code
);

CREATE TABLE player (
    id          BIGSERIAL PRIMARY KEY,
    team_id     BIGINT NOT NULL REFERENCES team(id),
    name        TEXT NOT NULL,
    number      INTEGER,
    position    TEXT,
    active      BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE match (
    id                      BIGSERIAL PRIMARY KEY,
    team_id                 BIGINT NOT NULL REFERENCES team(id),
    opponent                TEXT NOT NULL,
    match_date              TIMESTAMPTZ NOT NULL,
    home_away               TEXT CHECK(home_away IN ('home','away')),
    competition             TEXT,
    status                  TEXT CHECK(status IN ('scheduled','live','half_time','finished','locked')) NOT NULL DEFAULT 'scheduled',
    our_score               INTEGER NOT NULL DEFAULT 0,
    opponent_score          INTEGER NOT NULL DEFAULT 0,
    correction_window_ends  TIMESTAMPTZ,   -- set to finished_at + 1 day
    sheets_synced           BOOLEAN NOT NULL DEFAULT FALSE,
    coach_notes             TEXT    -- free-text coach/manager summary, entered post-match; input to the report generation engine
);

CREATE TABLE match_player (
    match_id     BIGINT NOT NULL REFERENCES match(id),
    player_id    BIGINT NOT NULL REFERENCES player(id),
    starting     BOOLEAN NOT NULL DEFAULT FALSE,
    is_sub       BOOLEAN NOT NULL DEFAULT FALSE,
    minute_on    INTEGER,
    minute_off   INTEGER,
    PRIMARY KEY (match_id, player_id)
);

CREATE TABLE event (
    id                    BIGSERIAL PRIMARY KEY,
    match_id              BIGINT NOT NULL REFERENCES match(id),
    minute                INTEGER NOT NULL,
    second_offset         INTEGER NOT NULL DEFAULT 0,   -- disambiguates ordering within the same minute
    event_type            TEXT NOT NULL,        -- 'goal', 'own_goal', 'goal_conceded', 'shot', 'foul', 'corner', 'card', 'sub', 'offside', 'save', 'penalty', etc.
    player_id             BIGINT REFERENCES player(id),  -- for 'save' and 'goal_conceded', this is the goalkeeper on the pitch at the time
    secondary_player_id   BIGINT REFERENCES player(id),
    metadata              JSONB,                 -- e.g. {"goal_type":"open_play","body_part":"right_foot"} or {"shot_type":"one_on_one"}
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted               BOOLEAN NOT NULL DEFAULT FALSE     -- soft delete, supports undo and audit trail
);

CREATE TABLE sheets_sync_log (
    id            BIGSERIAL PRIMARY KEY,
    match_id      BIGINT NOT NULL REFERENCES match(id),
    synced_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    status        TEXT CHECK(status IN ('success','failed')) NOT NULL,
    error_detail  TEXT
);

-- Recommended indexes given expected query patterns:
CREATE INDEX idx_event_match_id ON event(match_id) WHERE deleted = FALSE;
CREATE INDEX idx_match_team_id ON match(team_id);
```

**Design Notes:**
- The `metadata` field is `JSONB`, storing optional, variable-depth attributes and avoiding schema migrations as recording depth (Minimal/Normal/Detailed) increases, while still supporting native indexed queries against its contents if that becomes useful (e.g., "all goals scored from corners").
- Events are soft-deleted rather than hard-deleted, preserving an audit trail and supporting the undo/correction workflow. Statistics and report generation queries filter on `deleted = FALSE`.
- `correction_window_ends` is set to 24 hours after a match's finish time; a status check (executed on access or via a lightweight scheduled task) transitions `status` to `locked` once this window elapses, after which the API rejects further edits.
- `sheets_synced` and `sheets_sync_log` support the archival workflow described in Section 12, including retry handling for failed synchronization attempts.
- `goal_conceded` is distinct from `own_goal`: the former records an opponent's goal against the team while a specific goalkeeper is on the pitch (for goalkeeper statistics); the latter records the (rarer) case of a defensive player scoring into their own net. Both increment `opponent_score`.
- The current goalkeeper is derived from `match_player`/on-pitch state (the on-pitch player whose `position` is goalkeeper), consistent with how the "on-pitch roster" is derived for outfield event flows (see Section 5). This derived state is used to auto-attribute `save` and `goal_conceded` events to the correct goalkeeper without requiring manual selection in the common case.
- `coach_notes` is captured once, post-match, and is not part of the live event log. It is treated as unstructured input to the report generation engine (Section 8) and is never used as a source of statistical facts.
- Because all data access goes through the SQLAlchemy repository layer (Section 3.1), this schema — and the engine underneath it — can change again in the future with changes isolated to that layer.

---

## 5. Event Recording Interaction Model

### 5.1 Recording Depth Tiers
- **Minimal:** Goals, cards, and substitutions only.
- **Normal (default):** Adds shots, corners, fouls, and offsides.
- **Detailed:** Adds assists, shot outcomes, goal type/body part, foul context, and free-text notes.

The recording tier may be selected before or during a match. Every tier produces a valid report; higher tiers enable richer reports and statistics.

### 5.2 Match Screen (Operator View)
The interface presents large, fixed-position touch targets; the match clock and current score remain visible at all times. "Undo Last" and "End Match" are accessible without scrolling.

### 5.3 Event Flows (Summary)
- **Goal:** Select Goal → select scorer (limited to players currently on the pitch) → optionally select assist/type or skip → event recorded.
- **Substitution:** Select Sub → select outgoing player → select incoming player → event recorded; the "on pitch" roster used by all other event flows updates automatically.
- **Card:** Select Card → select Yellow/Red → select player → optionally record a reason → event recorded.
- **Foul / Corner / Shot / Offside:** Similar shallow flows (one to three selections), each supporting optional secondary detail.
- **Save:** Select Save → the current on-pitch goalkeeper is auto-attributed → optionally record shot difficulty/type or skip → event recorded. A manual goalkeeper override is available for the uncommon case where attribution should differ (e.g., a mid-play positional swap).
- **Goal Conceded:** Select Goal Conceded → opponent score increments automatically, the current on-pitch goalkeeper is auto-attributed for statistical purposes → optionally record scoring detail (e.g., opponent shirt number as free text, goal type) or skip → event recorded. This is distinct from the "Goal" flow, which records the team's own scoring.
- **Undo Last:** Always accessible; soft-deletes the most recent event and reverts derived state (score, on-pitch roster, current goalkeeper), broadcasting the change to connected viewers.
- **Post-Match Correction:** The event timeline supports tap-to-edit (player, time, deletion) until the correction window closes.

### 5.4 Match Setup (Pre-Match)
A team administrator creates a match record (opponent, date, home/away designation, competition) and selects the starting lineup and substitutes from the team's player roster, including designation of the starting goalkeeper. This establishes the "on pitch" state (including current goalkeeper) used to constrain player selection and auto-attribution during the match.

### 5.5 Match Completion (Post-Match)
Upon selecting "End Match," the operator or coach enters the final score (auto-populated from recorded goals/goals-conceded, editable) and may optionally enter free-text coach/manager notes summarizing their impression of the match. This step is optional and does not block match finalization; notes entered here are passed to the report generation engine described in Section 8.

---

## 6. Real-Time Update Design

- Each match is assigned a dedicated WebSocket channel.
- Any accepted state change (event creation, undo, score update) is broadcast to all clients subscribed to that match's channel, including the recording operator's own client, ensuring consistent state across all views.
- Public viewers subscribe in a read-only capacity; no authentication is required.
- On reconnection (e.g., following a device lock/unlock cycle), the client retrieves full current match state via REST prior to resubscribing to the WebSocket channel, avoiding the need for offline queuing or event replay logic.

---

## 7. API Design (Representative Endpoints)

| Method | Endpoint | Purpose | Authorization |
|---|---|---|---|
| POST | `/auth/login` | Exchange a team access code for a session token | None |
| POST | `/matches` | Create a new match | Team |
| POST | `/matches/{id}/lineup` | Configure starting lineup and substitutes | Team |
| POST | `/matches/{id}/start` | Transition match status to live | Team |
| POST | `/matches/{id}/events` | Record an event | Team |
| DELETE | `/matches/{id}/events/{event_id}` | Soft-delete an event (undo/correction) | Team |
| PATCH | `/matches/{id}/events/{event_id}` | Edit an event (within correction window) | Team |
| POST | `/matches/{id}/end` | Finalize match, record final score, initiate correction window | Team |
| POST | `/matches/{id}/notes` | Submit or update coach/manager notes (within correction window) | Team |
| GET | `/matches/{id}` | Retrieve full current match state | Public |
| GET | `/matches/{id}/report` | Retrieve generated match report (AI narrative, with deterministic fallback) | Public |
| POST | `/matches/{id}/report/regenerate` | Re-trigger AI narrative generation (e.g., after notes are edited) | Team |
| GET | `/teams/{id}/stats` | Retrieve season-to-date team statistics | Public |
| GET | `/players/{id}/stats` | Retrieve season-to-date player statistics | Public |
| GET/POST | `/admin/teams/{id}/access-code` | View/rotate a team's access code | Admin |
| WS | `/ws/matches/{id}` | Live event stream for a given match | Public (read-only) |

---

## 8. Report Generation Engine

Report generation follows a two-stage, hybrid approach: a deterministic fact-extraction stage, followed by a pluggable AI-assisted narrative stage. This preserves the principle of never fabricating information while producing a more natural, readable report than a pure template would allow, and keeps the choice of AI backend an implementation detail rather than an architectural commitment.

### 8.1 Stage 1 — Deterministic Fact Extraction
The system first assembles a structured, factual summary directly from the event log, independent of any AI component:
- Final score, scorers, and minutes.
- Assists, where recorded.
- Cards, substitutions, and their minutes.
- Shot, corner, foul, and goalkeeper statistics (saves, goals conceded), where recorded.
- This structured summary is itself sufficient to render a complete, correct, if plainly worded, report. It also serves as the guaranteed fallback report (see 8.4).

### 8.2 Stage 2 — Pluggable Narrative Generation
The structured summary from Stage 1, together with the optional free-text coach/manager notes (Section 5.5), is passed to a `NarrativeProvider` — an internal interface with a single method, conceptually `generate(facts, notes) -> text`. Two concrete implementations are anticipated:

- **`GeminiNarrativeProvider`** — calls the Google Gemini API.
- **`OllamaNarrativeProvider`** — calls a self-hosted Ollama instance (e.g., Llama 3.2 3B or Phi-4-mini), reached either on the same host or over a secure tunnel (Tailscale/Cloudflare Tunnel) to personally owned hardware.

Both implementations receive an identical grounding instruction: compose a natural-language report using only the facts provided, treating the coach's notes as color commentary/context rather than as a source of additional factual claims (e.g., scores, player names, or statistics not present in the structured summary). Because both implementations satisfy the same interface and receive the same inputs, the active provider is selected via a single configuration value (e.g., `NARRATIVE_PROVIDER=gemini|ollama|none`), and switching — or running both side-by-side during evaluation — requires no change to the API layer, the service layer, or the database.

The generated narrative is presented to the operator/coach for review before being finalized.

### 8.3 Extensibility: Adding a New Provider
Introducing a new narrative backend (a different hosted API, a different local model, or a future in-house model) requires only:
1. A new class implementing the `NarrativeProvider` interface.
2. A configuration entry pointing to it.

No changes to report request handling, the data model, or any other part of the system are required — consistent with the modularity principle established in Section 3.1.

### 8.4 Reliability & Fallback
Because report generation occurs after a match has ended (not during live recording), it is not on the critical path for in-match reliability. The fallback chain is: configured `NarrativeProvider` → Stage 1 deterministic summary. This covers multiple failure modes uniformly:
- The Gemini API is unavailable, rate-limited, or returns an error.
- A self-hosted Ollama instance is offline (e.g., home hardware powered down, tunnel disconnected) — a materially more likely occurrence than a managed API being unavailable, and one the architecture treats as an ordinary, expected condition rather than an edge case.
- In either case, the system falls back to the Stage 1 deterministic summary, ensuring a report is always available regardless of which provider is configured or its current availability.
- The AI-generated narrative can be regenerated on demand (e.g., after editing coach notes during the correction window, or after switching providers) without re-entering match data.
- Provider credentials/endpoints (API keys, tunnel addresses) are stored server-side only and never exposed to the frontend; all provider calls are made exclusively from the backend.

### 8.4 Output
The interface provides a "Copy Report" action producing plain text formatted for external use (e.g., pasting into TeamPlus), alongside the option to view or copy the underlying structured (Stage 1) summary. CSV/JSON export of the structured event data is a straightforward extension.

---

## 9. Statistics Engine

Statistics are computed on demand (or cached) from the event log, without requiring separate manual data entry:

- **Team-level:** matches played, win/draw/loss record, goals scored/conceded, per-match averages, shots, corners, fouls, cards, goal distribution by time interval and by type (derived from `metadata`).
- **Player-level:** matches, goals, assists, shots (including on-target), cards. Minutes-played-derived metrics (e.g., goals per 90 minutes) are deferred pending consistent population of substitution timing data.
- **Goalkeeper-level:** saves, goals conceded while on the pitch, save percentage (saves ÷ (saves + goals conceded)), and clean sheets (matches with zero goals conceded while on the pitch for the full match). These are derived from `save` and `goal_conceded` events attributed to the goalkeeper via the on-pitch/current-goalkeeper derivation described in Section 4.2.

These are computed via SQL aggregation over the `event` and `match_player` tables; a dedicated analytics pipeline is not required at the current scale.

---

## 10. Authentication & Security

- Each team is assigned a single access code (stored as a bcrypt hash); no individual user accounts are created.
- A successful login exchanges the access code for a short-lived, signed session token (JWT) scoped to the corresponding `team_id`.
- All write operations (event creation, editing, deletion; match creation and status transitions) require a valid session token whose `team_id` matches the target match's owning team.
- Read operations (match state, reports, statistics) are intentionally unauthenticated.
- Baseline protections include rate limiting on `/auth/login` to mitigate credential brute-forcing, request validation via Pydantic models, and parameterized queries (via SQLAlchemy) to prevent SQL injection.
- Access codes are rotated via the administrative interface described in Section 14.

---

## 11. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance | Event recording round-trip time under 300ms on typical mobile network conditions; UI updates should appear instantaneous via optimistic local updates reconciled with server responses |
| Availability | A single server instance is sufficient for MVP scale; high-availability infrastructure is not required |
| Device Support | Mobile-first responsive design; primary target is mid-range Android/iOS browsers in portrait orientation |
| Data Integrity | No hard deletion of events during or after a live match; soft-delete only, preserving undo and audit capability |
| Data Retention | The live database retains all historical data indefinitely (see Section 12); Google Sheets serves as a parallel, human-readable archive |
| Modularity | Database access, report narrative generation, and archival synchronization are each implemented behind internal interfaces (Section 3.1), so any one can be replaced or extended independently |
| Accessibility | Touch targets sized per WCAG guidance; high-contrast score display suitable for outdoor visibility |

---

## 12. Data Archival Strategy

### 12.1 Rationale
At the operating scale of this system (five teams, an estimated several dozen matches and several hundred events per half-season), the database does not require periodic clearing for performance or storage reasons — this is even more true on managed PostgreSQL than it would have been on a single SQLite file. The originally proposed six-month export-and-reset cycle was primarily a mechanism for producing a periodic archive; with continuous synchronization to Google Sheets, this forcing function is no longer necessary, and the live database can be retained indefinitely as the system's source of truth.

### 12.2 Synchronization Design
- Upon a match transitioning to `locked` status (i.e., the 24-hour correction window has elapsed), the system synchronizes that match's complete data to Google Sheets via the `ArchiveSyncProvider` interface (Section 3.1).
- Each team is represented by a dedicated, human-readable tab (or set of tabs) reflecting match logs and summary statistics, rather than a raw mirror of the underlying database tables.
- Synchronization occurs once per match (not per event), minimizing dependency on the Google Sheets API during live match recording and avoiding any impact on recording latency or reliability.
- Synchronization attempts are logged in `sheets_sync_log`; failures are retried, and `sheets_synced` is only set once synchronization succeeds.
- PostgreSQL remains the authoritative source of truth for all live application functionality (report generation, statistics, live match state); Google Sheets functions as a durable, accessible archive layer. Because synchronization is implemented behind the `ArchiveSyncProvider` interface, an additional or alternative archive destination could be added later without touching match-finalization logic.

---

## 13. Testing Strategy

- **Unit tests:** event-flow business logic (score updates, on-pitch roster derivation, report template generation, statistics aggregation).
- **Integration tests:** end-to-end API flows (match creation, lineup configuration, event recording, match finalization, report generation) against a test database.
- **WebSocket tests:** verification of correct broadcast behavior for goal, card, substitution, and undo events to all subscribed clients.
- **Manual acceptance testing:** validation against defined success criteria (see Appendix), including first-time usability of the goal-recording flow and correctness of reports generated from minimal-tier data.
- **Synchronization tests:** verification that Google Sheets archives accurately reflect database contents, and that failed synchronization attempts are retried rather than silently dropped.

---

## 14. Administrative Interface

A dedicated administrative interface will provide the following capabilities:
- Viewing and rotating team access codes.
- Viewing synchronization status/history for each team (via `sheets_sync_log`).
- Basic visibility into match statuses across all teams (scheduled, live, locked).

Detailed specification of this interface is addressed in the accompanying UI/UX Design Specification document.

---

## 15. Risks & Open Items

| Item | Description |
|---|---|
| Managed PostgreSQL free-tier limits | Free tiers (e.g., Neon) impose storage and compute ceilings; low likelihood of being reached at current scale, but worth monitoring as data accumulates over multiple seasons |
| Google Sheets API rate limits | To be validated against expected synchronization frequency (per-match, not per-event, mitigates this risk) |
| Concurrent write scaling | PostgreSQL comfortably supports current and substantially greater write volume than the five-team, amateur-scale usage requires; connection pooling can be introduced if this ever changes |
| Archive tab structure in Sheets | Final layout (per-team tabs, summary tab structure) to be finalized in the UI/UX Design Specification |
| Gemini API dependency | If selected as the active `NarrativeProvider`, report narrative generation depends on external API availability and incurs usage cost; mitigated by the deterministic Stage 1 fallback (Section 8.4), and by the call occurring post-match rather than during live recording |
| Self-hosted Ollama availability | If selected as the active `NarrativeProvider`, availability depends on personally owned hardware and tunnel connectivity remaining online; mitigated by the same Stage 1 fallback, since the provider architecture treats this as an expected, ordinary failure mode rather than a special case |
| Goalkeeper attribution edge cases | Auto-attribution assumes one designated goalkeeper on the pitch at a time; unusual scenarios (e.g., an outfield player temporarily in goal) rely on the manual override described in Section 5.3 |

---

## 16. Future Enhancements (Post-MVP)

- Advanced statistics: expected goals, possession, passing networks, tackles/interceptions.
- Visual analytics: shot maps, goal maps, form-over-time visualizations.
- AI-assisted report refinement, strictly bounded to existing recorded facts.
- Support for multiple simultaneous recorders per match, with backend-side merge by timestamp.
- Direct API integration with TeamPlus, contingent on API availability.
- Minutes-played-derived statistics (e.g., goals/assists per 90 minutes).
- Formation tracking and performance breakdowns by formation/position.
- Evaluation and potential adoption of newer or larger local models as a `NarrativeProvider`, should self-hosted inference quality need to improve without incurring hosted API costs.

---

## Appendix: MVP Scope Summary

**Pre-Match:** Match creation, opponent/competition/date entry, starting lineup and substitute selection.

**In-Match:** Goal, Goal Conceded, Shot, Foul, Corner, Card, Substitution, Offside, Save, and Undo actions; automatic clock and timestamp management; dynamic on-pitch roster and current-goalkeeper tracking.

**Post-Match:** Final score entry, optional coach/manager notes, event review/edit/delete within the correction window, AI-assisted report generation (with deterministic fallback), report export.

**Statistics (MVP):** Matches played, win/draw/loss record, goals scored/conceded, player goals/assists, cards, corners, shots, goalkeeper saves/goals conceded/save percentage/clean sheets.

**Archival:** Automatic, per-match synchronization to Google Sheets upon match lock; no periodic database reset required.
