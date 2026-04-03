# UnoArena — Requirements Document

> Proyecto académico · Arquitectura de Microservicios · ITBA 2026

---

## 1. Purpose

This document defines the functional and non-functional requirements for UnoArena, a real-time multiplayer platform for competitive Uno. It establishes the business capabilities, domain rules, and quality attributes that guide the domain design.

---

## 2. Global Scope

UnoArena is a real-time multiplayer platform for Uno that must support:

- Ad-hoc casual game rooms for small groups of players (2–10).
- Large-scale elimination tournaments (up to 1,000,000 players).
- Live game and tournament updates for players and spectators.
- Dual competitive ranking: global Elo for casual play and tournament-placement rating for tournament play.
- Authoritative validation of gameplay and results.
- Full auditability of competitive matches.

The platform must be the **authoritative source of truth** for gameplay state, tournament progression, and competitive outcomes.

**Constraints:**
- Academic project — not production.
- No external identity providers (no Keycloak, Auth0, etc.).
- Focus on requirements and design, not implementation.

---

## 3. Functional Requirements

### 3.1 Global Functional Requirements

- **FR-G1**: The system must allow registered users to participate as players and, when authorized, as spectators.
- **FR-G2**: The system must provide real-time visibility of relevant game and tournament state changes.
- **FR-G3**: The system must act as the authoritative source of truth for rule validation, game state, match state, and final outcomes.
- **FR-G4**: The system must ensure competitive fairness by rejecting invalid or unauthorized state transitions.
- **FR-G5**: The system must preserve consistency across gameplay, tournament progression, ranking updates, and audit history.

### 3.2 Room Play Context

**Room & Match Lifecycle:**

- **FR-R1**: The system must allow the creation of ad-hoc game rooms.
- **FR-R2**: A room must support a minimum of 2 players and a maximum of 10 players.
- **FR-R3**: The system must allow eligible players to join a room that is open for participation and not full.
- **FR-R4**: A room must have an explicit lifecycle with at least the states: waiting, in_progress, and completed.
- **FR-R5**: The system must define and enforce the conditions under which a match may start.
- **FR-R19**: A match within a room consists of a **best-of-three series** of individual games.

**Gameplay:**

- **FR-R6**: The system must maintain and expose the current turn owner and valid turn order.
- **FR-R7**: The system must validate whether a played card is legal according to the current game state.
- **FR-R8**: The system must support gameplay actions including play card, draw card, Uno call, challenge Uno call, and penalty resolution.
- **FR-R9**: The system must support time-sensitive rule interactions when timing affects move validity.
- **FR-R12**: The system must reject stale, invalid, or conflicting actions against the authoritative game state. Concurrent actions on the same game must be serialized using strict sequence numbers; stale actions must be rejected.
- **FR-R28**: All card shuffling and draws must be generated **server-side** using a deterministic, seeded RNG. No card distribution or random outcome may depend on client-side input.

**Uno Call Mechanics:**

- **FR-R10**: A player who plays their second-to-last card must call "Uno!" before the next player takes their turn.
- **FR-R11**: Any opponent may challenge a missed Uno call within a **5-second window** after the card is played. The window closes when the next player begins their turn, even if the 5 seconds have not elapsed.
- **FR-R21**: If a player fails to call "Uno!" and is successfully challenged, they draw **2 penalty cards**.
- **FR-R22**: If a player is challenged but did correctly call "Uno!", the **challenger** draws 2 penalty cards (false challenge penalty).

**Match Completion & Results:**

- **FR-R13**: The system must detect when a game is completed (a player plays their last card) and determine the game winner.
- **FR-R14**: The system must publish final match results (after the best-of-three series) to dependent business capabilities, including tournament progression and ranking.
- **FR-R23**: Upon match completion, the system must produce a **placement order** of all players in the room (1st through last), based on individual game wins within the match. This placement is used by the Ranking context for Elo calculation.

**Disconnection & Reconnection:**

- **FR-R15**: A disconnected player has a **60-second** reconnection window before being considered inactive.
- **FR-R16**: During the reconnection window, the disconnected player's turn is **skipped** (treated as a pass). No bot substitution occurs.
- **FR-R24**: If the reconnection window expires, an **automatic forfeit** is issued regardless of whose turn it is at that moment.
- **FR-R25**: Forfeit in a **casual room** ends the player's participation; the game continues with remaining players.
- **FR-R26**: Forfeit in a **tournament room** counts as a loss for that match; the player is eliminated from the tournament.
- **FR-R27**: A player who reconnects within the window resumes with their original hand intact.

**Timeouts & Abandoned Matches:**

- **FR-R17**: The system must define timeout rules for player turns.
- **FR-R18**: The system must define the business behavior for abandoned or stalled matches.

### 3.3 Tournament Context

- **FR-T1**: The system must allow tournament creation and configuration.
- **FR-T2**: The system must allow player registration during the permitted enrollment period.
- **FR-T3**: The system must support tournaments with up to 1,000,000 players.
- **FR-T4**: A tournament must have an explicit lifecycle with at least the states: planned, open_for_registration, in_progress, completed, and cancelled.
- **FR-T5**: The system must manage brackets and determine player grouping into rooms of up to 10 players for each round.
- **FR-T6**: Within each room, players play a three-game match. The **top 3 players** by match wins advance to the next round.
- **FR-T7**: The system must mark non-advancing players as eliminated.
- **FR-T8**: The system must expose tournament progress, including round status, bracket position, and advancement state.
- **FR-T9**: Tournament rounds proceed in elimination until **10 or fewer players** remain, at which point a single final room is created to determine the champion.
- **FR-T10**: Only valid and finalized match outcomes may affect tournament progression.
- **FR-T11**: The system must remain correct under massive simultaneous completion of tournament matches.
- **FR-T12**: The system must define seeding rules when tournament rules require seeded entry.
- **FR-T13**: Tie-breaking for advancement: (1) the player with the **lower cumulative card-point total** across the tied games advances; (2) if still tied, the player with the **earliest time of final-game completion** advances.
- **FR-T14**: The system must define the business behavior for tournament cancellation.
- **FR-T15**: The system must define the business behavior for interrupted tournament progression and recovery.

### 3.4 Ranking Context

- **FR-K1**: The system must maintain a **global Elo-based ranking** that applies to **casual (ad-hoc) rooms only**.
- **FR-K2**: Tournament play uses a **separate tournament-placement rating**, not the global Elo.
- **FR-K3**: Elo is updated **once per completed game** (not per match or per tournament).
- **FR-K4**: Elo delta is calculated from the **final placement order** within the room (1st through last).
- **FR-K5**: **Abandoned games** (forfeit by all remaining players) **do not affect Elo**.
- **FR-K6**: The system must preserve a history of rating changes over time.
- **FR-K7**: The system must expose current ranking information to authorized users.
- **FR-K8**: The system must prevent duplicate or repeated ranking updates for the same finalized result.

### 3.5 Spectator & Live View Context

- **FR-S1**: The system must allow eligible users to observe ongoing matches as spectators.
- **FR-S2**: Players and spectators must receive live updates about game state changes relevant to the game they are viewing.
- **FR-S3**: Users must be able to receive live tournament progression updates.
- **FR-S4**: Spectators must have read-only access and must not affect gameplay state.
- **FR-S5**: Live views must reflect the authoritative state and reconcile after rejected or outdated actions.
- **FR-S6**: Spectators must only see public information (player names, card counts, discard pile). Private hands must never be visible to spectators.

### 3.6 Audit & Game History Context

- **FR-A1**: The system must preserve an immutable history of game-relevant state changes. Every state change must be appended to the game log **before** being broadcast to clients.
- **FR-A2**: The system must trace all relevant player actions and system-generated rule resolutions to the corresponding game.
- **FR-A3**: The system must preserve sufficient information to reconstruct or replay game progression, including all RNG seeds used for shuffles and draws.
- **FR-A4**: The system must preserve traceability of card distribution and any random outcome used in authoritative play. Every random outcome must be deterministic and reproducible from the stored seed.
- **FR-A5**: The system must support post-match review for dispute resolution.
- **FR-A6**: The system must define retention rules for historical game and tournament records.
- **FR-A7**: The system must maintain audit logs for **sensitive operations** beyond gameplay (e.g., login attempts, session invalidations, role changes, tournament cancellations).

### 3.7 Identity & Session Context

- **FR-I1**: The system must authenticate users and manage session lifecycle.
- **FR-I2**: The system must enforce **single-active-session per player**: a new login invalidates the previous session.
- **FR-I3**: The system must support role-based access control (player, spectator, tournament operator, admin).
- **FR-I4**: Session expiry during an active match must trigger the disconnection grace timer flow.

---

## 4. Non-Functional Requirements

### 4.1 Performance

- **NFR-P1**: The system must provide low-latency propagation of accepted game state changes.
- **NFR-P2**: The system must support high volumes of concurrent active games and gameplay actions.
- **NFR-P3**: The system must tolerate burst traffic during tournament round starts and completions.
- **NFR-P4**: The system must support large numbers of concurrent players and spectators receiving live updates.

### 4.2 Scalability

- **NFR-SC1**: The system must scale horizontally as the number of players, games, tournaments, and spectators grows.
- **NFR-SC2**: The platform should allow independent scaling of gameplay, tournament, ranking, live view, and audit capabilities.
- **NFR-SC3**: The system must handle tournament scenarios with up to 1,000,000 participants without losing business correctness.

### 4.3 Consistency and Correctness

- **NFR-C1**: The platform must maintain a single authoritative version of game state at any point in time.
- **NFR-C2**: Actions affecting the same game must be processed in a valid business order.
- **NFR-C3**: Concurrent, stale, or conflicting actions must not corrupt authoritative state.
- **NFR-C4**: Tournament advancement must remain correct under mass simultaneous result processing.
- **NFR-C5**: Ranking updates must remain consistent with finalized competitive outcomes.

### 4.4 Reliability and Availability

- **NFR-R1**: The platform must remain highly available for gameplay, tournament progression, and live state consumption.
- **NFR-R2**: Failures in one business capability must not invalidate already accepted game results.
- **NFR-R3**: The system must support reliable recovery of authoritative gameplay and tournament state after interruption.
- **NFR-R4**: Historical game and audit records must be durably preserved.

### 4.5 Security and Fairness

- **NFR-SE1**: The system must protect against invalid or manipulated gameplay outcomes.
- **NFR-SE2**: Critical gameplay decisions and game state transitions must not depend on client-side authority.
- **NFR-SE3**: The system must enforce permissions for players, spectators, administrators, and tournament operators.
- **NFR-SE4**: The platform must provide sufficient traceability for fraud investigation and dispute handling.
- **NFR-SE5**: The system must enforce **multi-layer rate limits** (per IP, per user, per room/tournament action) with adaptive throttling to prevent abuse and flooding.
- **NFR-SE6**: The system must validate all inputs at system boundaries and apply signed event integrity where needed for sensitive operations.
- **NFR-SE7**: The system must enforce single-active-session per player to prevent session takeover and concurrent manipulation.

### 4.6 Maintainability and Evolvability

- **NFR-M1**: The solution must preserve clear bounded context separation between gameplay, tournaments, ranking, live views, and auditing.
- **NFR-M2**: Game rules, tournament rules, and ranking policies should be evolvable without redefining unrelated domains.
- **NFR-M3**: The platform must use consistent domain terminology across requirements and domain interactions.
- **NFR-M4**: The solution should be extensible to support future tournament formats, ranking policies, and gameplay variants.

---

## 5. Domain Rules & Constraints

- **DR-1**: A room must contain between 2 and 10 players.
- **DR-2**: A completed game must not accept further gameplay actions.
- **DR-3**: Only the authoritative platform state may determine game results and tournament advancement.
- **DR-4**: A player may advance in a tournament only if recognized as a top-3 finisher of the current round's room.
- **DR-5**: A spectator may observe but must not modify competitive state. Spectators must never see private player hands.
- **DR-6**: Global Elo rankings may only be modified from valid, finalized casual game results. Abandoned games do not affect Elo.
- **DR-7**: Authoritative game history must be immutable once committed. State changes must be logged before being broadcast to clients (log-before-broadcast invariant).
- **DR-8**: A player may not take an action outside their valid turn unless the rules explicitly allow it (e.g., Uno challenge within the 5-second window).
- **DR-9**: A stale action must not override a newer authoritative state.
- **DR-10**: A tournament match result must not be applied more than once to advancement or ranking.
- **DR-11**: A cancelled tournament must not continue advancing players after cancellation becomes effective.
- **DR-12**: A disconnected player remains part of the game. Their turns are skipped (passed) during the 60-second grace window. Forfeit occurs only on window expiry.
- **DR-13**: A match consists of a best-of-three series of games. The match winner is the first player to win 2 games.
- **DR-14**: Tournament rounds continue until 10 or fewer players remain, at which point a final room determines the champion.
- **DR-15**: Each player may have only one active session at a time. A new login invalidates the previous session.
- **DR-16**: All card shuffling, draws, and random outcomes must be generated server-side with a deterministic seeded RNG. The seed must be stored per game for audit replay.
