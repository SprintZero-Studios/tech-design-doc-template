# Creature Clash: Duel Loop Prototype: Technical design document

<!-- A filled example of TECH-DESIGN.md: the engineering brief from a real
     Murmur run, published with the owner's permission. Text is as the run
     wrote it; only the layout follows this template. Full run:
     https://www.trymurmur.studio/examples/creature-clash -->

## Overview

A private, single-player web prototype for validating a stat-category duel loop before any art or public release is built. There are no other players, no accounts, and no cloud sync in scope, so the architecture is deliberately local-first: a Next.js client running entirely in the founder's browser, with all creature, battle, and playtest data stored in an on-device SQLite database. Nothing about the duel, the world, or the AI trainers depends on a server, so no backend API is built. The one thing this project must get right is the turn-resolution logic for the stat duel (type triangle + abilities + growth), so that component is designed first and in depth; everything else (world navigation, roster growth, gym/badge progression, AI opponents, telemetry) is built around feeding it clean inputs and recording its outputs for balance analysis.

## Data flow

On launch, the Next.js client loads the Creature & Content Data Layer and the founder's save file from local SQLite. The player navigates the route/gym map (World Route Navigation); encountering a wild creature or trainer starts a Battle row. Each round, the player picks a category in the UI while the AI Trainer Decision Engine picks for the opponent; both choices go into the Duel Battle Engine, which applies the type chart and any ability effects, resolves HP deltas, and appends a BattleTurn row. When a battle ends, Creature Roster & Growth awards XP and prompts stat allocation (written as an idempotent XPAllocation); if it was a deciding gym battle, Gym & Badge Progression issues a Badge and runs the idempotent GymSwap. All writes land in the local SQLite database — nothing is sent to a server. Playtest Telemetry logs the session and the founder periodically exports the SQLite file/JSON to review against the success metrics.

## Components

| Component | Responsibility |
| --- | --- |
| Duel Battle Engine | The core of the product. Inputs per round: the two active creatures (stats after growth allocation, current HP, ability, elemental type), the category each side picks that round (attack/defense/speed/special), and a seeded RNG value for the battle. Outputs: which side wins the round, the HP delta applied, whether an ability fired and what it changed, and an immutable BattleTurn log row. Category choices are revealed simultaneously, the type-triangle multiplier is applied, abilities resolve in a fixed documented order (steal > force > negate-disadvantage > other), and the same battle_id, rng_seed and choice sequence always reproduce the same outcome, so balance bugs are reproducible. |
| Ability Rules Registry | Defines the launch set of unique creature abilities as structured trigger/effect data (steal-category, force-category, negate-type-disadvantage, etc.) that the Battle Engine reads and resolves; keeps rule-bending logic declarative and testable rather than hard-coded per creature. |
| AI Trainer Decision Engine | Deterministic, rule-based (non-LLM) heuristics that choose a category for every AI-controlled opponent — wild creatures, route trainers, and gym leaders — e.g. counter the player's last category, lean into a type advantage, or protect a low-HP creature. Kept deterministic so duel outcomes stay reproducible for balance testing. |
| Creature Roster & Growth | Tracks each owned CreatureInstance's stats, HP, and ability; awards XP after every battle (win, lose, or draw) and exposes the trainer-directed allocation UI that lets the player assign earned XP to any stat category, writing an idempotent XPAllocation record per battle per creature. |
| Gym & Badge Progression | Models the ordered sequence of gyms, each with a type theme and a leader trainer; issues a Badge on a 3v3 gym win and runs the post-gym creature swap (player gives one creature, receives one of the leader's), recorded as a single idempotent GymSwap transaction keyed to the deciding battle. |
| World Route Navigation & Wild Catching | Renders the persistent, walkable route/gym map, resolves wild encounters from each route's weighted species table, and handles the starter-selection and catch flow that adds a new CreatureInstance to the roster. |
| Creature & Content Data Layer | Loads and serves the placeholder/real creature species, base stats, types, and ability definitions used for prototyping (per the founder's stated strategy of validating the loop before original art); isolated behind one module so the real dataset can later be swapped for original creatures without touching the Battle Engine. |
| Local Persistence & Save Layer | Owns the on-device SQLite database (schema below): reads/writes all game state, exposes save export/import as a file so the founder can back up a playtest run manually, since no cloud sync is in scope. |
| Playtest Telemetry & Leaderboard | Logs PlaytestSession records (duration, self-reported 1–5 fun score, battles played) and derives the four success metrics (completion rate, session frequency, fun score, category variety) plus a simple badge-count leaderboard against the tracked AI trainers, all computed locally from the SQLite tables. |

## Data models

### Trainer

| Field | Type |
| --- | --- |
| id | uuid |
| name | text |
| role | text |
| gymId | uuid \| null |
| decisionProfile | text \| null |
| createdAt | timestamp |

**Relationships**

- Trainer has many CreatureInstance (as owner)
- Trainer has many Badge (as earner, for leaderboard)
- Trainer 1:1 Gym (as leader)

### ElementType

| Field | Type |
| --- | --- |
| id | text |
| name | text |

**Relationships**

- ElementType referenced by Species, Gym, Route, TypeEffectiveness

### TypeEffectiveness

| Field | Type |
| --- | --- |
| attackerType | text |
| defenderType | text |
| multiplier | numeric |

**Relationships**

- Composite key (attackerType, defenderType)

### Ability

| Field | Type |
| --- | --- |
| id | uuid |
| name | text |
| description | text |
| triggerType | text |
| effectType | text |
| effectParams | jsonb |

**Relationships**

- Ability referenced by Species.abilityId and CreatureInstance.abilityId
- Ability referenced by BattleTurn.abilityTriggeredId

### Species

| Field | Type |
| --- | --- |
| id | uuid |
| name | text |
| primaryType | text |
| baseAttack | int |
| baseDefense | int |
| baseSpeed | int |
| baseSpecial | int |
| abilityId | uuid |
| catchRate | numeric |
| isLeaderExclusive | boolean |
| sourceNote | text |

**Relationships**

- Species has many CreatureInstance
- Species has many RouteEncounter

### CreatureInstance

| Field | Type |
| --- | --- |
| id | uuid |
| speciesId | uuid |
| ownerTrainerId | uuid |
| nickname | text \| null |
| level | int |
| xpTotal | int |
| attackStat | int |
| defenseStat | int |
| speedStat | int |
| specialStat | int |
| currentHp | int |
| maxHp | int |
| abilityId | uuid |
| teamSlot | int \| null |
| caughtAt | timestamp \| null |
| createdAt | timestamp |

**Relationships**

- CreatureInstance belongs to Species and Trainer
- CreatureInstance referenced by BattleTurn, XPAllocation, GymSwap

### Gym

| Field | Type |
| --- | --- |
| id | uuid |
| name | text |
| typeTheme | text |
| leaderTrainerId | uuid |
| sequenceOrder | int |
| badgeName | text |

**Relationships**

- Gym has one leader Trainer
- Gym has many Badge
- Gym has many GymSwap via Battle

### Badge

| Field | Type |
| --- | --- |
| id | uuid |
| trainerId | uuid |
| gymId | uuid |
| earnedAt | timestamp |

**Relationships**

- Badge belongs to Trainer and Gym
- unique (trainerId, gymId)

### Route

| Field | Type |
| --- | --- |
| id | uuid |
| name | text |
| sequenceOrder | int |
| typeTheme | text \| null |

**Relationships**

- Route has many RouteEncounter
- Route referenced by Battle.routeId

### RouteEncounter

| Field | Type |
| --- | --- |
| id | uuid |
| routeId | uuid |
| speciesId | uuid |
| encounterWeight | numeric |

**Relationships**

- RouteEncounter belongs to Route and Species

### Battle

| Field | Type |
| --- | --- |
| id | uuid |
| kind | text |
| routeId | uuid \| null |
| gymId | uuid \| null |
| playerTrainerId | uuid |
| opponentTrainerId | uuid \| null |
| rngSeed | bigint |
| winnerSide | text \| null |
| startedAt | timestamp |
| endedAt | timestamp \| null |
| idempotencyKey | text |

**Relationships**

- Battle has many BattleTurn
- Battle has 0..1 GymSwap
- Battle has many XPAllocation

### BattleTurn

| Field | Type |
| --- | --- |
| id | uuid |
| battleId | uuid |
| turnNumber | int |
| playerCreatureId | uuid |
| opponentCreatureId | uuid |
| playerCategory | text |
| opponentCategory | text |
| typeMultiplierPlayer | numeric |
| typeMultiplierOpponent | numeric |
| abilityTriggeredId | uuid \| null |
| hpDeltaPlayer | int |
| hpDeltaOpponent | int |

**Relationships**

- BattleTurn belongs to Battle
- unique (battleId, turnNumber)
- source data for category-choice-variety metric

### XPAllocation

| Field | Type |
| --- | --- |
| id | uuid |
| battleId | uuid |
| creatureInstanceId | uuid |
| xpEarned | int |
| allocatedStat | text \| null |
| allocatedAmount | int \| null |
| allocatedAt | timestamp \| null |
| idempotencyKey | text |

**Relationships**

- XPAllocation belongs to Battle and CreatureInstance

### GymSwap

| Field | Type |
| --- | --- |
| id | uuid |
| battleId | uuid |
| playerCreatureGivenId | uuid |
| leaderCreatureReceivedId | uuid |
| swappedAt | timestamp |

**Relationships**

- GymSwap belongs to Battle (1:1)

### PlaytestSession

| Field | Type |
| --- | --- |
| id | uuid |
| startedAt | timestamp |
| endedAt | timestamp \| null |
| durationSec | int \| null |
| funScore | int \| null |
| battlesPlayedCount | int |
| notes | text \| null |

**Relationships**

- Standalone; aggregated for success-metric reporting

## Schema

```sql
CREATE TABLE type_effectiveness (
  attacker_type text NOT NULL REFERENCES element_types(id),
  defender_type text NOT NULL REFERENCES element_types(id),
  multiplier numeric NOT NULL,
  PRIMARY KEY (attacker_type, defender_type)
);

CREATE TABLE abilities (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  trigger_type text NOT NULL,
  effect_type text NOT NULL,
  effect_params jsonb
);

CREATE TABLE trainers (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  role text NOT NULL CHECK (role IN ('player','ai_route','gym_leader'))
);
```

## Tech stack

| Layer | Choice and why |
| --- | --- |
| Frontend | Next.js (App Router) as a local-first single-page client; no SSR or multi-tenant needs for a solo private prototype. |
| Backend | No server/API layer. All game, duel, and progression logic runs client-side in the browser — this is a single local player with no accounts or multiplayer, so a backend would add infrastructure the PRD explicitly excludes. |
| Database | Client-side SQLite via sql.js (WASM), persisted to the browser's OPFS/IndexedDB on the founder's own device. No cloud database, so no game or playtest data is ever transmitted anywhere — matches the PRD's explicit 'no cloud sync' non-goal. |
| Auth | Not required. Single local player, no account system per PRD non-goals; the app opens straight into the on-device save file. |
| LLM | Not required. AI trainers use deterministic, rule-based heuristics (e.g. counter the player's last category, lean into a type advantage, protect low-HP creatures) rather than a generative model, so battle outcomes stay reproducible for the seeded-RNG replay requirement and the 'category-choice variety' success metric. All stat math, type-triangle multipliers, XP, and swap logic are deterministic code, never model output. |
| Infra | No cloud infra required to run the prototype; launched via local dev/build on the founder's machine. Optional unlisted Vercel static deploy only to test on a phone browser — it would serve JS/asset bundles only, never game data, keeping the 'no public release' non-goal intact. |
| Dev tools | TypeScript, ESLint/Prettier as standard; Vitest unit tests specifically around the Duel Battle Engine's round-resolution function, since its correctness and determinism are what the whole validation effort depends on. |

## Engineering tasks

1. **Local schema & persistence setup:** Stand up the sql.js (SQLite/WASM) database from the schema above inside the browser, with OPFS/IndexedDB-backed storage, a migration runner, and a manual save-file export/import so a multi-week playtest isn't lost to browser storage eviction.
2. **Access setup (no-op, single local profile):** Confirm the app boots directly into one local save with no login screen, and add a guard/lint rule preventing any accidental network or cloud-service calls from creeping into the client, per the no-accounts/no-cloud-sync non-goals.
3. **Local run & deploy setup:** Configure `next dev`/local static build as the primary way to run the prototype; wire an optional unlisted Vercel deploy for phone-browser testing that ships client assets only, with no data endpoints.
4. **Duel Battle Engine core:** Implement the deterministic round-resolution algorithm: category reveal, type-triangle multiplier lookup, fixed-order ability resolution, stat comparison, HP delta, and BattleTurn logging, backed by a seeded RNG and a full Vitest suite covering type/ability edge cases.
5. **Ability Rules Registry & launch ability set:** Build the trigger/effect data model for abilities (steal-category, force-category, negate-type-disadvantage) and author the initial roster's abilities against it, decoupled from the Battle Engine's core loop.
6. **AI Trainer Decision Engine:** Implement rule-based category-selection heuristics for wild encounters, route trainers, and gym leaders, with distinct difficulty/decision profiles per Trainer.decisionProfile.
7. **Creature Roster & XP Growth flow:** Build the XP-earning logic for every battle outcome (win/lose/draw) and the trainer-directed stat-allocation UI, writing one idempotent XPAllocation row per battle per creature.
8. **Gym/Badge progression & 3v3 swap:** Implement gym sequencing, badge issuance on a 3v3 gym win, and the post-gym creature swap as a single idempotent GymSwap transaction keyed to the deciding battle.
9. **World route navigation & wild catching:** Build the persistent route/gym map, weighted wild-encounter resolution from RouteEncounter data, starter selection, and the catch flow that creates a new CreatureInstance.
10. **Playtest telemetry & metrics export:** Log PlaytestSession records (duration, 1-5 fun score, battles played) and add an export view that computes the four PRD success metrics directly from the local SQLite tables.

## Open engineering questions

- How many gyms/badges constitute a 'complete' V1 story mode for playtesting purposes? (from PRD)
- What threshold of fun/completion data should trigger investment in original creature art? (from PRD)
- Should XP-driven stat allocation be fully free-form per category, or guided with suggested distributions? (from PRD)
- Which existing creature dataset will be used for prototyping, and what legal exposure exists even for a private, unreleased build? (from PRD)
- What is the exact round-resolution/damage formula and tie-break rule when both sides pick the same category or land equal adjusted stats? (this doc assumed 'difference in stat value, no damage on tie' — needs founder sign-off)
- When more than one ability could trigger in the same round, what is the intended precedence beyond the placeholder steal > force > negate order used here?
- Is browser OPFS/IndexedDB persistence reliable enough across a 4+ week playtest, or should the prototype be wrapped in a thin desktop shell (e.g. a Tauri build) for a guaranteed on-disk SQLite save file?
- Is the prototype meant to be played on a phone browser (touch input, small screen) during validation, or is desktop-only acceptable for this phase?
