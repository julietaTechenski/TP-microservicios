---
title: UnoArena — Architecture & Design Document

---

# UnoArena — Architecture & Design Document

> Derived from: `UnoArena_Global_Real-Time_Uno_Platform.pdf` and `Requisitos_AM_UNO_DDD.md`  
> Status: Draft v3.0 — Added tactical DDD artifacts (Aggregates, Entities, Value Objects), Hexagonal Architecture per service, subdomain classification, and Ubiquitous Language glossary.

---

## 1. Purpose

This document defines the architecture and design of UnoArena based on the domain requirements established in the DDD specification. Its purpose is to translate business rules, bounded contexts, and quality attributes into a concrete technical design that guides implementation decisions.

It describes how the platform is decomposed into services, how responsibilities are assigned across bounded contexts, how domain models are represented through tactical DDD patterns, and how the system ensures correctness, scalability, real-time responsiveness, and auditability.

This document serves as the bridge between requirements and implementation, providing a shared architectural reference for development, validation, and future evolution of the platform.

---

## 2. Global Scope

UnoArena is a real-time multiplayer platform for Uno that must support:

- ad-hoc game rooms for small groups of players,
- large-scale elimination tournaments,
- live game and tournament updates for players and spectators,
- competitive ranking based on match outcomes,
- authoritative validation of gameplay and results,
- full auditability of competitive matches.

The platform must be the **authoritative source of truth** for gameplay state, tournament progression, and competitive outcomes.

---

## 3. Bounded Contexts

### 3.1 Room Play Context
Responsible for ad-hoc room creation, player participation, turn progression, card actions, penalties, and match lifecycle.

### 3.2 Tournament Context
Responsible for tournament registration, bracket progression, round lifecycle, eliminations, and final standings.

### 3.3 Ranking Context
Responsible for player ratings, global ranking, and rating history.

### 3.4 Spectator & Live View Context
Responsible for live state visibility for players and spectators.

### 3.5 Audit & Game History Context
Responsible for immutable game history, replayability, and dispute support.

---
## 4. Functional Requirements

### 4.1 Global Functional Requirements

- **FR-G1**: The system must allow registered users to participate as players and, when authorized, as spectators.
- **FR-G2**: The system must provide real-time visibility of relevant game and tournament state changes.
- **FR-G3**: The system must act as the authoritative source of truth for rule validation, match state, and final outcomes.
- **FR-G4**: The system must ensure competitive fairness by rejecting invalid or unauthorized state transitions.
- **FR-G5**: The system must preserve consistency across gameplay, tournament progression, ranking updates, and audit history.

### 4.2 Room Play Context

- **FR-R1**: The system must allow the creation of ad-hoc game rooms.
- **FR-R2**: A room must support a minimum of 2 players and a maximum of 10 players.
- **FR-R3**: The system must allow eligible players to join a room that is open for participation and not full.
- **FR-R4**: A room must have an explicit lifecycle with at least the states: waiting, in progress, and completed.
- **FR-R5**: The system must define and enforce the conditions under which a match may start.
- **FR-R6**: The system must maintain and expose the current turn owner and valid turn order.
- **FR-R7**: The system must validate whether a played card is legal according to the current match state.
- **FR-R8**: The system must support gameplay actions including play card, draw card, Uno call, and penalty resolution.
- **FR-R9**: The system must support time-sensitive rule interactions when timing affects move validity.
- **FR-R10**: The system must support the declaration of Uno when a player reaches the required state.
- **FR-R11**: The system must detect and resolve missed Uno declarations according to the game rules.
- **FR-R12**: The system must reject stale, invalid, or conflicting actions against the authoritative room state.
- **FR-R13**: The system must detect when a match is completed and determine the winner.
- **FR-R14**: The system must publish final match results to dependent business capabilities, including tournament progression and ranking.
- **FR-R15**: The system must define the business behavior for player disconnection during an active match.
- **FR-R16**: The system must define the business behavior for player reconnection to an active match.
- **FR-R17**: The system must define timeout rules for player turns.
- **FR-R18**: The system must define the business behavior for abandoned or stalled matches.

### 4.3 Tournament Context

- **FR-T1**: The system must allow tournament creation and configuration.
- **FR-T2**: The system must allow player registration during the permitted enrollment period.
- **FR-T3**: The system must support tournaments with up to 1,000,000 players.
- **FR-T4**: A tournament must have an explicit lifecycle with at least the states: planned, open for registration, in progress, completed, and cancelled.
- **FR-T5**: The system must manage brackets and determine player grouping into matches for each round.
- **FR-T6**: The system must determine winners of each match and advance them correctly.
- **FR-T7**: The system must mark non-advancing players as eliminated.
- **FR-T8**: The system must expose tournament progress, including round status, bracket position, and advancement state.
- **FR-T9**: The system must detect tournament completion and determine the final winner and standings.
- **FR-T10**: Only valid and finalized match outcomes may affect tournament progression.
- **FR-T11**: The system must remain correct under massive simultaneous completion of tournament matches.
- **FR-T12**: The system must define seeding rules when tournament rules require seeded entry.
- **FR-T13**: The system must define tie-breaking behavior when tournament rules require tie resolution.
- **FR-T14**: The system must define the business behavior for tournament cancellation.
- **FR-T15**: The system must define the business behavior for interrupted tournament progression and recovery.

### 4.4 Ranking Context

- **FR-K1**: The system must maintain a global competitive ranking of players.
- **FR-K2**: The ranking model must support Elo-based rating updates.
- **FR-K3**: Ranking updates must be triggered only from valid, finalized match results.
- **FR-K4**: The system must preserve a history of rating changes over time.
- **FR-K5**: The system must expose current ranking information to authorized users.
- **FR-K6**: Tournament match outcomes must affect rankings when business rules allow.
- **FR-K7**: The system must prevent duplicate or repeated ranking updates for the same finalized result.

### 4.5 Spectator & Live View Context

- **FR-S1**: The system must allow eligible users to observe ongoing matches as spectators.
- **FR-S2**: Players and spectators must receive live updates about match state changes relevant to the game they are viewing.
- **FR-S3**: Users must be able to receive live tournament progression updates.
- **FR-S4**: Spectators must have read-only access and must not affect gameplay state.
- **FR-S5**: Live views must reflect the authoritative state and reconcile after rejected or outdated actions.

### 4.6 Audit & Game History Context

- **FR-A1**: The system must preserve an immutable history of game-relevant state changes.
- **FR-A2**: The system must trace all relevant player actions and system-generated rule resolutions to the corresponding match.
- **FR-A3**: The system must preserve sufficient information to reconstruct or replay match progression.
- **FR-A4**: The system must preserve traceability of card distribution and any random outcome used in authoritative play.
- **FR-A5**: The system must support post-match review for dispute resolution.
- **FR-A6**: The system must define retention rules for historical match and tournament records.

---
## 5. Non-Functional Requirements

### 5.1 Performance

- **NFR-P1**: The system must provide low-latency propagation of accepted game state changes.
- **NFR-P2**: The system must support high volumes of concurrent active matches and gameplay actions.
- **NFR-P3**: The system must tolerate burst traffic during tournament round starts and completions.
- **NFR-P4**: The system must support large numbers of concurrent players and spectators receiving live updates.

### 5.2 Scalability

- **NFR-SC1**: The system must scale horizontally as the number of players, matches, tournaments, and spectators grows.
- **NFR-SC2**: The platform should allow independent scaling of gameplay, tournament, ranking, live view, and audit capabilities.
- **NFR-SC3**: The system must handle tournament scenarios with up to 1,000,000 participants without losing business correctness.

### 5.3 Consistency and Correctness

- **NFR-C1**: The platform must maintain a single authoritative version of match state at any point in time.
- **NFR-C2**: Actions affecting the same match must be processed in a valid business order.
- **NFR-C3**: Concurrent, stale, or conflicting actions must not corrupt authoritative state.
- **NFR-C4**: Tournament advancement must remain correct under mass simultaneous result processing.
- **NFR-C5**: Ranking updates must remain consistent with finalized competitive outcomes.

### 5.4 Reliability and Availability

- **NFR-R1**: The platform must remain highly available for gameplay, tournament progression, and live state consumption.
- **NFR-R2**: Failures in one business capability must not invalidate already accepted match results.
- **NFR-R3**: The system must support reliable recovery of authoritative gameplay and tournament state after interruption.
- **NFR-R4**: Historical match and audit records must be durably preserved.

### 5.5 Security and Fairness

- **NFR-SE1**: The system must protect against invalid or manipulated gameplay outcomes.
- **NFR-SE2**: Critical gameplay decisions and match state transitions must not depend on client-side authority.
- **NFR-SE3**: The system must enforce permissions for players, spectators, administrators, and tournament operators.
- **NFR-SE4**: The platform must provide sufficient traceability for fraud investigation and dispute handling.

### 5.6 Maintainability and Evolvability

- **NFR-M1**: The solution must preserve clear bounded context separation between gameplay, tournaments, ranking, live views, and auditing.
- **NFR-M2**: Game rules, tournament rules, and ranking policies should be evolvable without redefining unrelated domains.
- **NFR-M3**: The platform must use consistent domain terminology across requirements and domain interactions.
- **NFR-M4**: The solution should be extensible to support future tournament formats, ranking policies, and gameplay variants.

---

## 6. Domain Rules & Constraints

- **DR-1**: A room must contain between 2 and 10 players.
- **DR-2**: A completed match must not accept further gameplay actions.
- **DR-3**: Only the authoritative platform state may determine match results and tournament advancement.
- **DR-4**: A player may advance in a tournament only if recognized as a valid winner of the current stage.
- **DR-5**: A spectator may observe but must not modify competitive state.
- **DR-6**: Rankings may only be modified from valid, finalized competitive results.
- **DR-7**: Authoritative game history must be immutable once committed.
- **DR-8**: A player may not take an action outside their valid turn unless the rules explicitly allow it.
- **DR-9**: A stale action must not override a newer authoritative state.
- **DR-10**: A tournament match result must not be applied more than once to advancement or ranking.
- **DR-11**: A cancelled tournament must not continue advancing players after cancellation becomes effective.
- **DR-12**: A disconnected player remains part of the match until domain rules determine timeout, reconnection, or forfeiture.

---

## 7. Architecture Overview

UnoArena is structured as a set of independently deployable services aligned with the six bounded contexts defined in the requirements. Communication between contexts flows through an asynchronous event bus (Kafka), while clients interact via REST for commands and Server-Sent Events (SSE) for live state.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│          REST (commands)          SSE (state updates)            │
└─────────────────┬─────────────────────────┬─────────────────────┘
                  │                         │
         ┌────────▼────────┐    ┌───────────▼──────────┐
         │   API Gateway   │    │    SSE Broadcaster    │
         │  + Auth (JWT)   │    │  (stateless, scaled)  │
         └────────┬────────┘    └───────────▲──────────┘
                  │                         │ Redis Streams
                  │              ┌──────────┴──────────┐
                  │              │  Projection Service  │
                  │              │  (player + spectator │
                  │              │   channels)          │
                  │              └──────────▲──────────┘
      ┌───────────▼──────────────────────────────────────────────┐
      │                     Kafka Event Bus                       │
      └───┬──────────┬─────────────┬──────────────┬─────────────┘
          │          │             │              │
   ┌──────▼──┐ ┌─────▼──────┐ ┌───▼─────┐ ┌────▼──────────┐
   │  Room   │ │ Tournament │ │ Ranking │ │ Audit/History │
   │  State  │ │Orchestrator│ │ Service │ │   Service     │
   │  Pods   │ └─────┬──────┘ └─────────┘ └───────────────┘
   └─────────┘       │
                ┌────▼───────┐
                │  Bracket   │
                │ CQRS Read  │
                │   Model    │
                └────────────┘

   ┌──────────────────────────┐
   │   Identity Service       │
   │  (auth, sessions, roles) │
   └──────────────────────────┘
        ▲ consulted by API Gateway, SSE Broadcaster, Room Pods
```

---

## 8. Subdomain Classification & Bounded Context to Service Mapping

### 8.1 Subdomain Classification

| Bounded Context | Subdomain Type | Justification |
|---|---|---|
| Room Play | **Core Domain** | The competitive game engine is UnoArena's primary differentiator. It contains the highest business complexity (turn logic, concurrency, RNG fairness, penalty resolution) and demands the most rigorous DDD modeling. |
| Tournament | **Supporting Domain** | Essential for the platform's value proposition but not the core competitive logic itself. Bracket management and orchestration support the core game experience. Warrants a rich model but with less investment than Room Play. |
| Ranking | **Supporting Domain** | The Elo system adds competitive depth but follows well-known algorithms. Important for user engagement, not a differentiator in itself. |
| Spectator & Live View | **Supporting Domain** | Enables the viewing experience that supports the core gameplay. Primarily a projection and delivery concern with moderate domain logic (view filtering, eligibility). |
| Audit & Game History | **Generic Domain** | Standard append-only logging and replay infrastructure. No unique business logic — any event sourcing / audit system would behave similarly. |
| Identity & Session | **Generic Domain** | Authentication, token lifecycle, and RBAC are commodity functions. Built in-house per project constraints (no external services), but the domain logic is industry-standard. |

### 8.2 Service Mapping

| Bounded Context (DDD) | Service(s) | Primary Storage | Scalability Unit |
|---|---|---|---|
| Room Play | Room State Pod (per room) | In-memory + event log | Pod per room |
| Tournament | Tournament Orchestrator + Bracket CQRS | PostgreSQL + Redis/DynamoDB | Horizontal |
| Ranking | Ranking Service | PostgreSQL | Horizontal |
| Spectator & Live View | SSE Broadcaster + Redis Streams + Spectator Projection Service | Redis Streams | Stateless horizontal |
| Audit & Game History | Audit Service | Append-only store (e.g., Kafka compacted topic or S3) | Horizontal |
| Identity & Session | Identity Service | PostgreSQL + Redis (token cache) | Horizontal |

---

## 9. Room Play Context Design

### 9.1 Room State Machine

Each room runs as an isolated state-machine pod in Kubernetes. The lifecycle is:

```
waiting ──► in_progress ──► completed
   │                            │
   └──── cancelled ◄────────────┘
```

Transitions are driven by explicit Kafka events. A room in `completed` or `cancelled` state must reject all further gameplay actions (DR-2).

### 9.2 Turn & Move Processing

- All REST commands (`POST /rooms/{id}/play`, `POST /rooms/{id}/draw`, `POST /rooms/{id}/uno`) carry a **sequence number** representing the client's known state version.
- The room-state pod validates the sequence number against its authoritative state. If stale, it returns `HTTP 409 Conflict` (FR-R12, NFR-C3).
- The client reconciles by consuming the SSE stream to obtain the current authoritative state.

### 9.3 RNG & Card Distribution

- A dedicated, seeded **RNG Service** handles all shuffles and draws server-side (FR-A4, NFR-SE2).
- Each random outcome is recorded in the immutable game log *before* being broadcast.
- The seed is stored and associated with each match, enabling full replay and audit (FR-A3).

### 9.4 Uno Call & Penalty Resolution

- The "Uno!" call is a time-sensitive action. The room-state pod records server-side timestamps for all card play events.
- **Challenge window** (DR-17): the window opens when the server records the second-to-last card play event and closes when the server records the first valid action of the subsequent turn. Only challenges received within this server-evaluated window are accepted. Client-side timing is not authoritative.
- If a player reaches 1 card without calling Uno and a valid challenge is submitted within the window, the penalty (draw 2 cards) is applied server-side and recorded in the game log before the next turn proceeds (FR-R11).
- DR-8 is enforced: a player cannot act outside their turn unless the rules explicitly allow it (e.g., jump-in variant).

### 9.5 Disconnection & Reconnection

- A disconnected player remains in the match (DR-12). The room-state pod starts a **reconnection grace timer** of 60 seconds (configurable per room type, competitive vs. casual) (FR-R15, FR-R16).
- **Turn timing is independent of disconnection** (DR-18): the standard turn timeout continues to run while a player is disconnected. If the turn timeout expires, the pod applies the configured timeout action (auto-draw or skip) and advances the turn. The grace timer only governs whether the player remains a match participant.
- On reconnection within the grace period, the pod replays the delta from the player's last known sequence number via SSE, restoring full game state.
- If the grace timer expires without reconnection, the pod applies the configured abandonment action (forfeit or continued auto-pilot) per FR-R18, and emits a `player.forfeited` event to Kafka.
- Session expiry mid-match (FR-G6) triggers the same disconnection flow: the Identity Service notifies the room-state pod, which starts the grace timer.

### 9.6 Match Completion & Result Publishing

When a winner is detected (FR-R13):
1. The room-state pod emits a `room.completed` event to Kafka with the authoritative `GameResult`.
2. The pod transitions to `completed` and rejects further actions.
3. Downstream consumers (Tournament Orchestrator, Ranking Service, Audit Service) react to this event independently (FR-R14, NFR-R2).

### 9.7 Tactical Model — Aggregates, Entities & Value Objects

**Aggregate Root: `Room`** — The transactional consistency boundary for all gameplay within a single match. All state mutations (play card, draw, call Uno, apply penalty) are coordinated through the Room aggregate. No external service may modify room state directly.

Entities within the Room aggregate:

- `Player` — Identified by `PlayerId`. Holds the player's hand, connection status, and Uno-call flag. Mutable throughout the match lifecycle (cards added/removed, status changes). Identity persists across reconnections.
- `Turn` — Identified by `TurnNumber`. Represents the current active turn with its owner, start timestamp, and timeout deadline. A new Turn entity is created on each turn advancement.

Value Objects:

- `Card` — Immutable. Defined by `Color` (red, yellow, green, blue, wild) and `Face` (0–9, skip, reverse, draw-two, wild, wild-draw-four). Two cards with the same color and face are interchangeable (structural equality).
- `GameResult` — Immutable snapshot produced at match completion. Contains the winner's `PlayerId`, final scores per player, match duration, and total turns played. Published to Kafka as part of `room.completed`.
- `SequenceNumber` — Monotonically increasing integer. Self-validating: must be non-negative and strictly greater than the previous value. Used for optimistic concurrency control on every client command.
- `RngSeed` — Immutable. The seed value used to initialize the deterministic RNG for a specific match. Stored in the game log for replay and audit.
- `ChallengeWindow` — Immutable time interval defined by an open timestamp and a close timestamp, both server-authoritative. Determines validity of Uno call challenges.
- `GraceTimer` — Duration value (default 60s) representing the reconnection window. Configurable per room type (competitive vs. casual).
- `RoomConfiguration` — Immutable after room creation. Encapsulates player count (2–10), room type (casual/competitive), turn timeout duration, and grace timer duration.

Invariants enforced by the Room aggregate:

- A card can only be played if it is in the active player's hand and matches the discard pile top card by color or face (or is a wild).
- Only the current turn owner may submit play/draw actions (DR-8), unless jump-in variant rules apply.
- The sequence number on every command must match the room's current sequence number; otherwise the command is rejected.
- A room in `completed` or `cancelled` state rejects all gameplay commands (DR-2).
- The Uno challenge is only valid within the server-defined `ChallengeWindow` (DR-17).

## 10. Tournament Context Design

### 10.1 Tournament Lifecycle

```
planned ──► open_for_registration ──► in_progress ──► completed
                                             │
                                         cancelled
```

Cancellation stops all further bracket advancement immediately (DR-11, FR-T14).

### 10.2 Bracket Management at 1M Scale

- The Tournament Orchestrator manages bracket grouping and round progression.
- Each tournament defines a fixed **players-per-match** value at creation (FR-T16, DR-16). This value is between 2 and 10 and applies to every round. Example capacity impact:
  - 1,000,000 players, 10 per match → **100,000 concurrent rooms** in round 1
  - 1,000,000 players, 2 per match → **500,000 concurrent rooms** in round 1
- The orchestrator batches room creation requests to Kubernetes using a token-bucket rate limiter to avoid thundering-herd overload. A burst of 100,000 room creations must be fully provisioned within 30 seconds (aligned with NFR-P3).
- Bracket state and advancement are persisted in PostgreSQL (write side). A **CQRS read model** (Redis or DynamoDB) serves all bracket visualization queries (FR-T8, FR-T17). The read model is eventually consistent with a maximum lag of 5 seconds under normal conditions (FR-T17). This lag is acceptable for standings and bracket views; it does not affect authoritative advancement decisions, which are always made against the write side.

### 10.3 Result Ingestion & Idempotency

- The orchestrator consumes `room.completed` events from Kafka.
- **Idempotency** (DR-14): before processing any result, the orchestrator checks the composite key `{matchId, eventType, sequenceNumber}` against a processed-events log. Duplicate or replayed events are silently discarded without side effects. All other consumers (Ranking Service, Audit Service) enforce the same idempotency contract.
- **Cancellation guard** (DR-13): if the tournament's write-side state is `cancelled` at the time a `room.completed` event is processed, the event is discarded. Bracket advancement, winner notification, and ranking updates are not triggered. The orchestrator emits a `result.discarded.cancelled_tournament` event for audit purposes.
- All advancement steps follow a **saga pattern**: score reporting → winner advancement → next-round room creation. Each step is idempotent and replayable on failure (FR-T15).

### 10.4 Seeding & Tie-Breaking

- Seeding rules (FR-T12) and tie-breaking behavior (FR-T13) are configurable per tournament. They are implemented as pluggable strategies in the orchestrator, supporting evolvability (NFR-M2, NFR-M4).

### 10.5 Interrupted Progression & Recovery

- If the orchestrator crashes mid-advancement, recovery relies on the idempotency guarantees defined in §5.3. On restart, the orchestrator re-reads unprocessed `room.completed` events from Kafka and replays the saga steps. Since each step is idempotent (DR-14), duplicate execution produces no side effects.

### 10.6 Tactical Model — Aggregates, Entities & Value Objects

**Aggregate Root: `Tournament`** — The consistency boundary for all tournament lifecycle operations: registration, bracket generation, round advancement, and cancellation. The Tournament aggregate ensures that bracket state is never corrupted by concurrent result ingestion.

Entities within the Tournament aggregate:

- `Bracket` — Identified by `BracketId`. Represents the full elimination structure. Contains the tree of matches organized by round. Mutated as results arrive and winners advance.
- `Match` — Identified by `MatchId`. Links a set of `PlayerSlot` entries to a specific `RoomId`. Tracks match status (pending, in_progress, completed) and the resulting winner.

Value Objects:

- `PlayerSlot` — Immutable reference to a player within a bracket position. Contains `PlayerId` and seeding rank.
- `RoundNumber` — Non-negative integer identifying a tournament round. Self-validating: must be sequential.
- `TournamentConfiguration` — Immutable after creation. Encapsulates players-per-match (2–10), seeding strategy, tie-breaking strategy, and `partial-result ranking` flag.
- `SeedingStrategy` — Enum/strategy identifier (e.g., random, elo-based, manual). Determines initial bracket placement.
- `TieBreakingStrategy` — Enum/strategy identifier. Determines resolution when match results are ambiguous.

Invariants:

- A tournament in `cancelled` state discards all incoming `room.completed` events (DR-13).
- Bracket advancement only occurs on the write side (PostgreSQL), never on the CQRS read model.
- The players-per-match value is fixed at creation and immutable across all rounds (DR-16).
- A match cannot advance a winner unless it has a `completed` result.


---

## 11. Ranking Context Design

- The Ranking Service subscribes to `room.completed` events tagged as competitive (FR-K3, FR-K6).
- Before applying any Elo update, the service checks: (a) idempotency via DR-14, and (b) whether the originating tournament has reached `completed` state (DR-15). Results from cancelled tournaments are not applied to Elo.
- Tournaments with `partial-result ranking` enabled (configured at creation per DR-15) are an exception: their match results are ranking-eligible regardless of final tournament state.
- It applies **Elo rating** updates per result. The formula weights are configurable per game mode.
- Rating updates are idempotent on composite key `{matchId, eventType, sequenceNumber}` (DR-14) to prevent double-counting (FR-K7, DR-6).
- Rating history is preserved as an append-only log per player (FR-K4).
- The global ranking table is a read-optimized materialized view, refreshed asynchronously (FR-K5, NFR-P1).

### 11.1 Tactical Model — Aggregates, Entities & Value Objects

**Aggregate Root: `PlayerRanking`** — The consistency boundary for a single player's rating history. Each Elo update is applied through this aggregate, which enforces idempotency and maintains the append-only rating log.

Value Objects:

- `EloRating` — Immutable. Represents a player's rating at a point in time. Contains the numeric rating value and the timestamp of calculation.
- `EloUpdate` — Immutable. Represents a single rating change. Contains `MatchId`, previous rating, new rating, delta, and the game mode weight applied.
- `IdempotencyKey` — Composite value `{matchId, eventType, sequenceNumber}`. Used to deduplicate incoming events (DR-14).
- `RankingEligibility` — Value indicating whether a match result qualifies for Elo update. Encapsulates the check against tournament cancellation status and `partial-result ranking` flag (DR-15).

Invariants:

- An `EloUpdate` with a duplicate `IdempotencyKey` is silently discarded.
- Results from cancelled tournaments without `partial-result ranking` are never applied.
- Rating history is append-only; past entries are never modified.

---

## 12. Spectator & Live View Context Design

### 12.1 SSE Fan-Out Architecture

```
Room State Pod ──► Projection Service ──► Redis Streams ──► SSE Broadcaster ──► Clients
                    (player view)          (per-player channel)
                    (spectator view)       (spectator channel)
```

- Room state pods publish raw authoritative state changes to an internal Kafka topic after every accepted state transition.
- The **Projection Service** consumes these events and produces two distinct Redis Streams channels per room:
  - A **per-player channel** for each active player: includes that player's private hand.
  - A **spectator channel**: includes only public state (card counts per player, discard pile top card, current turn, direction, scores).
- The SSE Broadcaster tier is **stateless**: it holds long-lived HTTP connections and forwards patches from the appropriate channel based on the subscriber's role. It scales independently of game logic pods (NFR-SC2, NFR-P4).

### 12.2 Spectator Projection Definition

The spectator projection (FR-S6, DR-5) exposes the following fields and withholds the rest:

| Field | Spectator Visible | Rationale |
|---|---|---|
| Current turn owner | ✅ Yes | Public game state |
| Turn direction | ✅ Yes | Public game state |
| Discard pile top card | ✅ Yes | Public game state |
| Card count per player | ✅ Yes | Public game state |
| Draw pile size | ✅ Yes | Public game state |
| Uno call status per player | ✅ Yes | Public game state |
| Each player's hand (card values) | ❌ No | Private — withheld from all non-owning observers |
| RNG seed | ❌ No | Internal audit only |

### 12.3 Spectator Eligibility

Any registered user may spectate a public room or tournament match (FR-S7). Private rooms and private tournaments require explicit authorization from the room or tournament creator. Authorization is validated by the Identity Service before the SSE connection is established.

### 12.4 State Reconciliation

- If a client misses patches (e.g., after reconnection), it requests a snapshot from the Projection Service for its role (player or spectator) and then replays deltas from the corresponding Redis Stream.
- Rejected actions do not generate patches; the client's SSE stream inherently carries the corrected authoritative state (FR-S5).

### 12.5 Tactical Model — Aggregates, Entities & Value Objects

This context is projection-oriented; it does not own authoritative game state. Its domain model reflects read-side concerns.

**Aggregate Root: `RoomProjection`** — Represents the projected view of a room's current state, maintained per room. It is rebuilt from events emitted by the Room Play context.

Value Objects:

- `PlayerView` — The player-specific projection: includes the player's private hand, card count, and Uno-call status.
- `SpectatorView` — The public-only projection: card counts per player, discard pile top card, current turn, direction, draw pile size, Uno-call statuses. Private data (hands, RNG seed) is excluded (DR-5).
- `StatePatch` — An incremental delta describing a single state transition. Serialized and pushed to Redis Streams for SSE delivery.
- `SubscriberRole` — Enum: `player` or `spectator`. Determines which Redis Streams channel the SSE Broadcaster reads from.

Invariants:

- A `SpectatorView` must never contain any player's hand contents or the RNG seed.
- A `PlayerView` for player X must never contain player Y's hand contents.

---

## 13. Audit & Game History Context Design

- Every state change emitted by the room-state pod (card played, draw, Uno call, penalty, color change) is appended to an **immutable game log** before broadcast (FR-A1, FR-A2, DR-7).
- The log includes: event type, timestamp, player ID, sequence number, RNG seed reference (FR-A4).
- The Audit Service consumes these events from Kafka and persists them to an append-only store.
- Match logs support full replay (FR-A3) and dispute review (FR-A5).
- Retention rules (FR-A6) are configurable: competitive matches may have longer retention than casual rooms.

### 13.1 Tactical Model — Aggregates, Entities & Value Objects

**Aggregate Root: `GameLog`** — Identified by `MatchId`. Represents the complete, immutable record of a single match. The aggregate only supports append operations; no entries can be modified or deleted within the retention window.

Value Objects:

- `GameLogEntry` — Immutable. Contains event type, server timestamp, `PlayerId`, `SequenceNumber`, and `RngSeed` reference. Each entry represents a single atomic state change.
- `RetentionPolicy` — Immutable. Defines the retention duration based on match type (2 years for competitive, 90 days for casual per FR-A6).

Invariants:

- Entries are append-only; no mutation or deletion is permitted within the retention period.
- Every entry must have a valid sequence number that is strictly monotonic within its `GameLog`.
---

## 14. Identity & Session Context Design

- The Identity Service is the authority for player authentication and session lifecycle (FR-G6, BC 4.6).
- It issues short-lived bearer tokens (recommended TTL: 15 minutes) with refresh tokens for long-lived sessions.
- The API Gateway validates bearer tokens on every REST request. The SSE Broadcaster validates tokens at connection establishment.
- **Session expiry during an active match**: when a token expires and cannot be refreshed (e.g., refresh token revoked), the Identity Service publishes a `session.expired` event to Kafka. The room-state pod for any room that player is in receives this event and initiates the disconnection grace timer (DR-12, FR-G6) — identical to a network disconnection.
- Role claims (player, spectator, tournament operator, admin) are embedded in the token. Authorization for spectating private matches is resolved by the Identity Service at SSE connection time using room/tournament visibility settings.
- Redis is used as a token revocation cache to enable sub-second revocation without requiring a database round-trip on every request.

### 14.1 Tactical Model — Aggregates, Entities & Value Objects

**Aggregate Root: `UserAccount`** — The consistency boundary for a single user's identity, credentials, and role assignments. All authentication and session operations flow through this aggregate.

Entities within the UserAccount aggregate:

- `Session` — Identified by `SessionId`. Represents an active authenticated session. Tracks creation time, last activity, and associated refresh token. Mutable (can be refreshed or revoked).

Value Objects:

- `BearerToken` — Immutable. Short-lived JWT containing `UserId`, role claims, and expiration timestamp. TTL: 15 minutes.
- `RefreshToken` — Immutable. Long-lived opaque token used to obtain new bearer tokens without re-authentication.
- `RoleClaim` — Enum set: `player`, `spectator`, `tournament_operator`, `admin`. Embedded in the bearer token for authorization decisions.
- `Credentials` — Immutable. Encapsulates hashed password and salt. Never exposed outside the aggregate boundary.

Invariants:

- A revoked refresh token cannot be used to issue new bearer tokens.
- Role claims in a bearer token must reflect the user's current roles at issuance time.
- Session expiry during an active match triggers a `session.expired` event to Kafka (DR-12).

---

## 15. Cross-Cutting Concerns

### 15.1 Security & Fairness

- All game state authority resides server-side (NFR-SE2). Clients are untrusted input sources.
- Role-based access control enforces player / spectator / admin / tournament operator permissions (NFR-SE3), enforced via the Identity Service and bearer token role claims.
- The RNG service and immutable game log together provide the traceability required for fraud investigation (NFR-SE4).

### 15.2 Consistency Guarantees

- Each room-state pod is the **single authoritative writer** for its room (NFR-C1).
- Sequence numbers ensure ordered processing of actions per room (NFR-C2).
- Kafka consumers (tournament, ranking, audit) enforce idempotency using the composite key `{matchId, eventType, sequenceNumber}` per DR-14 (NFR-C4, NFR-C5).
- Tournament bracket advancement decisions are always made against the PostgreSQL write side, never the eventually consistent CQRS read model.

### 15.3 Observability

- Every Kafka event carries a `correlationId` traceable from REST input through to audit log.
- Metrics collected: action rejection rate (HTTP 409s), SSE connection count per channel type (player vs. spectator), Kafka consumer lag per consumer group, room lifecycle durations, Elo update throughput, challenge window hit/miss rate, grace timer expiry rate.

---

## 16. Ubiquitous Language Glossary

This glossary defines the authoritative meaning of domain terms across UnoArena. Where a term has a different meaning in different bounded contexts, the context-specific definition is noted.

| Term | Definition | Owning Context(s) |
|---|---|---|
| **Room** | A single game instance with 2–10 players, running as an isolated state-machine pod. Has a lifecycle: waiting → in_progress → completed/cancelled. | Room Play |
| **Match** | In Room Play: synonymous with a room's gameplay session. In Tournament: a specific bracket slot linking players to a room, tracked for advancement purposes. | Room Play, Tournament |
| **Player** | In Room Play: an active participant in a room, holding a hand of cards and a connection status. In Ranking: a rated entity with an Elo history. In Identity: a role claim on a user account. | Room Play, Ranking, Identity |
| **Turn** | The active period during which a single player may submit actions (play, draw, call Uno). Bounded by a timeout deadline. | Room Play |
| **Card** | An immutable game piece defined by a Color and a Face. Two cards with identical Color and Face are interchangeable. | Room Play |
| **Hand** | The ordered set of Cards currently held by a Player. Private to the owning player; never exposed to spectators or other players. | Room Play, Spectator |
| **Discard Pile** | The shared stack of played cards. Only the top card is relevant for determining play legality. | Room Play |
| **Draw Pile** | The shared deck from which players draw. Managed server-side by the RNG service. | Room Play |
| **Sequence Number** | A monotonically increasing integer representing the authoritative version of a room's state. Every client command must carry the current sequence number; stale values trigger HTTP 409. | Room Play |
| **Challenge Window** | A server-defined time interval during which other players may challenge a missing Uno call. Opens on the second-to-last card play event; closes on the first valid action of the next turn. | Room Play |
| **Grace Timer** | The reconnection window (default 60s) after a player disconnects. Turn timeouts continue independently. If the timer expires, the player forfeits or enters auto-pilot. | Room Play |
| **Game Result** | An immutable snapshot of a completed match: winner, scores, duration, turns played. Published as part of the `room.completed` event. | Room Play, Tournament, Ranking |
| **Tournament** | A multi-round elimination competition with a fixed bracket structure. Has a lifecycle: planned → open_for_registration → in_progress → completed/cancelled. | Tournament |
| **Bracket** | The elimination tree structure of a tournament. Organized by rounds. Only mutated through the write side (PostgreSQL). | Tournament |
| **Round** | A set of concurrent matches within a tournament. All matches in a round must complete before the next round begins. | Tournament |
| **Player Slot** | A position within a bracket assigned to a specific player, including their seeding rank. | Tournament |
| **Elo Rating** | A numeric skill rating for a player, updated after competitive matches using a configurable weighted formula. | Ranking |
| **Elo Update** | A single rating change event, linked to a specific match result. Idempotent on the composite key `{matchId, eventType, sequenceNumber}`. | Ranking |
| **Ranking Eligibility** | The determination of whether a match result qualifies for Elo adjustment. Depends on tournament completion status and `partial-result ranking` flag. | Ranking |
| **Spectator** | A user observing a room without participating. Receives only the public projection (SpectatorView). Must be authorized for private rooms/tournaments. | Spectator, Identity |
| **Player View** | The SSE projection sent to an active player: includes their private hand plus all public state. | Spectator |
| **Spectator View** | The SSE projection sent to spectators: public state only (card counts, discard pile, turn info). No hands, no RNG seed. | Spectator |
| **State Patch** | An incremental delta representing a single state transition, serialized to Redis Streams for SSE delivery. | Spectator |
| **Game Log** | The immutable, append-only record of every state change in a match. Supports replay and dispute resolution. | Audit |
| **Game Log Entry** | A single record in the game log: event type, timestamp, player ID, sequence number, RNG seed reference. | Audit |
| **Bearer Token** | A short-lived JWT (15-min TTL) containing user identity and role claims. Validated on every REST request and at SSE connection establishment. | Identity |
| **Refresh Token** | A long-lived opaque token used to obtain new bearer tokens without re-authentication. Revocable. | Identity |
| **Role Claim** | A permission level embedded in the bearer token: player, spectator, tournament_operator, or admin. | Identity |
| **Session** | An authenticated user session. Expiry during an active match triggers the disconnection grace timer flow. | Identity, Room Play |
| **Idempotency Key** | The composite value `{matchId, eventType, sequenceNumber}` used by all Kafka consumers to deduplicate events (DR-14). | Cross-cutting |
| **Correlation ID** | A unique identifier attached to every Kafka event, traceable from the originating REST request through to the audit log. | Cross-cutting |

---
