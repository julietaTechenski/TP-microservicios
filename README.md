# UnoArena

UnoArena is a real-time competitive Uno platform that supports both casual ad-hoc rooms (2–10 players) and massive elimination tournaments of up to 1,000,000 players. Players submit moves via REST and receive live game and tournament updates via Server-Sent Events. The platform handles split-second mechanics like Uno calls and challenges, a global Elo ranking system, and tournament bracket progression across potentially hundreds of thousands of simultaneous matches.

This repository contains the **domain design checkpoint** for the Microservices Architecture course (ITBA 2026), produced using Domain-Driven Design (DDD) and EventStorming as the primary discovery methodology.

## Contents

| File | Description |
|---|---|
| [REQUIREMENTS.md](./REQUIREMENTS.md) | Functional and non-functional requirements. Source of truth for traceability. |
| [DESIGN.md](./DESIGN.md) | Full DDD design: EventStorming, bounded contexts, aggregates, domain events, narrative flows, edge cases, and consistency strategy. |

## Domain Flow (summary)

```
Player / Operator
    → CreateRoom / CreateTournament
    → JoinRoom / RegisterForTournament
    → StartMatch → [best-of-three games] → MatchResultPublished
                                                ↓
                                    Tournament: top-3 advancement
                                    Ranking: Elo update (casual only)
                                    Audit: immutable game log
                                    Spectator: live projections
```

## Bounded Contexts

| Context | Role | Primary Responsibility |
|---|---|---|
| **Room Play** | Core Domain | Authoritative gameplay, Uno rules, match lifecycle |
| **Tournament** | Core Domain | Brackets, round advancement, champion determination |
| **Ranking** | Supporting | Global Elo (casual) and tournament-placement rating (separate) |
| **Spectator & Live View** | Supporting | Real-time public projections; hand privacy enforced at construction |
| **Audit & Game History** | Supporting | Immutable event log, traceability for dispute resolution |
| **Identity & Session** | Generic | Authentication, single active session per player, role control |
