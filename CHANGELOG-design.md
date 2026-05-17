# Design Changelog

This file records every change made to the domain-design package (`DESIGN.md`, `REQUIREMENTS.md`) for the Architecture Checkpoint. For each entry:

- **Artifact** names the file and section changed.
- **Deliverable** cites the corresponding Design Checkpoint deliverable (§5, 1–8).
- **Why** states the architecture or integration constraint that required the change.
- **Non-negotiable preserved?** confirms no Design Checkpoint invariant was weakened or dropped.

---

| # | Artifact changed | Deliverable | Why | Non-negotiable preserved? |
|---|---|---|---|---|
| 1 | `DESIGN.md` §2.3.1 — Spectator boundary treatment: stated that privacy is enforced at **projection construction time**, not delivery time. | Deliverable 6 — Consistency strategy and edge cases | Architecture builds `SpectatorView` by filtering private events before writing to the projection store, so a delivery-side authorization bug cannot leak hand data (`ARCHITECTURE.md` §5.4). | Yes. The privacy guarantee (spectators never see hands) is preserved and architecturally hardened. |
| 2 | `DESIGN.md` §8.2 — Open question Q3 (tournament-placement rating formula): noted that the architecture isolates the formula to `ranking-service` internals with no cross-context contract dependency. | Deliverable 8 — Open questions and assumptions | Architecture decision isolates the formula change surface to one service, reducing cross-context contract risk when the formula is eventually defined. | N/A. Open question, not a non-negotiable. |
| 3 | `DESIGN.md` §1.1 (EventStorming "After GameCompleted"), §3.1 (invariants), §4.1 (Room Creation to Completion narrative steps 7–8), §7.10 (design decision), §8.1 assumption A7 — match series changed from **always-3-games** to **best-of-three with early termination at 2 wins**. `REQUIREMENTS.md` FR-T6 updated from "three-game match" to "best-of-three match (early termination at two wins)". | Deliverables 3, 5, 7, 8 — Aggregates; Domain flows; Design decisions; Open questions and assumptions | The architecture assignment explicitly requires "early termination at two wins" (assignment §2, match-series coordination invariant). The prior design decision to always play 3 games directly contradicts that invariant. `REQUIREMENTS.md` FR-R19 ("best-of-three series") implies early termination under standard terminology; the ambiguity with FR-T6 ("three-game match") is resolved in favour of FR-R19. | Yes. Top-3 advancement, tie-breaking (game wins → cumulative card-point total → earliest completion time), and Elo-per-game are all unchanged; the tie-breaking rules are game-count agnostic. |
| 4 | `DESIGN.md` §3.1 internal events catalog — `GameCompleted`: replaced `isAbandoned (boolean)` with `outcome ∈ {completed, abandoned}`. `DESIGN.md` §3.1 integration events catalog — `GameResultPublished`: same rename on the `GameResult` VO field. `DESIGN.md` §3.1 integration events catalog — `MatchResultPublished`: added `outcome ∈ {completed, abandoned, forfeit_all}` to the `MatchResult` VO. `DESIGN.md` §4.3 and §7.11: updated references from `isAbandoned = true` to `outcome = abandoned`. `ARCHITECTURE.md` aligned to the same field names throughout. | Deliverable 4 — Commands and domain events catalog | An enum-style `outcome` field is more explicit and readable in the event log than a boolean, and is extensible (e.g. `forfeit_all` on `MatchResultPublished` distinguishes all-players-abandon from normal completion, which tournament-service needs to record the result as an advancement loss). No domain rule changed. | Yes. Elo is still skipped for `outcome = abandoned` games (DR-6). Tournament forfeits still count as losses for advancement. The distinction between abandoned and completed outcomes is preserved and now more explicitly typed. |

---

## Package-level affirmation

No Design Checkpoint non-negotiable domain guarantee was weakened or dropped by any of the above changes. Specifically:

- **Elo scope** (no tournament games, no abandoned casual games): enforced via `outcome = abandoned` check at ranking-service and the producer-side filter that publishes `GameResultPublished` only for casual rooms (entries 1, 4).
- **Tournament advancement** (top-3, best-of-three series, tie-break): preserved; early termination (entry 3) is consistent with standard best-of-three semantics and the assignment's stated invariant.
- **Single-active-session**: unchanged.
- **Log-before-broadcast**: unchanged; the outbox transaction is the architectural home of this invariant.
- **Spectator privacy**: hardened from delivery-time to construction-time enforcement (entry 1).
- **Sequence-number enforcement**: unchanged.
- **60-s reconnection and 5-s Uno! windows**: unchanged in duration and semantics.
