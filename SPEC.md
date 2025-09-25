# Quantum Tic-Tac-Toe — SPEC.md

> **Status:** Draft

## Overview

This document is the authoritative spec-as-code for the Quantum Tic-Tac-Toe project. It is intended to be stored at the root of a Git repository as SPEC.md and versioned alongside code and tests. Follow the commit discipline and small-iteration policy described below.

Purpose: produce a small, test-driven, playable implementation of Quantum Tic-Tac-Toe that demonstrates basic quantum concepts (superposition, collapse, objective vs. opponent-collapse), supports two collapse modes (original AJP 2006 opponent-choice and an objective automatic resolver), includes a minimal web UI with a plugin-able animation system, and is organized for easy incremental commits.

---

## Top-level deliverables

1. SPEC.md (this document)
2. Repo skeleton with src/, tests/, animations/, docs/, logs/ and CI placeholder (GitHub Actions) files.
3. Executable playable prototype (FastAPI backend + minimal HTML/JS frontend) — small enough to run locally.
4. Unit & integration tests (pytest) encoding the game's rules and collapse mechanics.
5. CHANGELOG.md and logs/prompts.log (prompt log; only user prompts appended).
6. Small, incremental commits suitable for clear git history and occasional isolated refactor commits.

---

## High-level game rules (behavioral spec)

The goal is a faithful, testable implementation of a Quantum Tic-Tac-Toe variant inspired by AJP 2006, with the following essential mechanics:

* Board: 3×3 squares indexed 1–9 (instead of 0–8), numbered like a telephone keypad.
  * The index digit appears in the lower-right corner of each square, smaller than the spooky mark but still legible.
  * The spooky mark (superposition, entanglement, etc.) remains the main visual element in the square.
* Players: X and O (X goes first).
* Quantum move (placement): a player places a quantum marker spanning two distinct squares (e.g., X: A↔B). This creates a link between the two squares labelled with the player's token and a move index (to disambiguate multiple links).
* Classical move (collapse): when collapse occurs, certain squares become classical (definitively X or O) rather than superposed.
* Cycle/cyclic entanglement: a cycle is a set of quantum links that form a closed loop across squares (graph cycle). Cycles are the common trigger for collapse.
* Collapse modes (UI toggle):
  * Opponent-choice (AJP 2006) — Default. When a cycle is created by player P's latest quantum placement, the other player (not P) chooses how the cycle collapses (which branches become classical). This choice is a strategic move (collapse move).
  * Objective/Automatic — Engine applies a deterministic collapse algorithm (seedable RNG optional) when a cycle is created. This is useful for deterministic demos and automated tests.
* Collapse behavior: collapse resolves conflicting superpositions such that squares contained by selected branches become classical. Implementation must match tests described in tests/.
* Win condition: classical three-in-a-row (row, column, diagonal). If no classical winner and board is classical-full (or no further moves), it's a draw.
* State save/load: game state serializable to JSON for reproducibility.

Note: the implementation should preserve the two-type-of-move flavor: placement (quantum link creation) vs collapse (resolution) — in opponent-choice mode a player sometimes is forced to play a collapse move immediately after the opponent's placement.

---

## Architecture & code organization

/ (repo root)
- SPEC.md
- README.md
- CHANGELOG.md
- logs/
  - prompts.log
- src/
  - app.py                  # FastAPI entrypoint (or small server)
  - engine/
    - __init__.py
    - game_engine.py       # GameEngine orchestration
    - board.py             # Board model & serialization
    - quantum_move.py      # QuantumMove model
    - resolvers.py         # CollapseResolver interface + implementations
    - ai_player.py         # Optional minimal AI
  - ui/
    - static/
      - index.html
      - main.js
      - styles.css
    - animations/          # animation plugin stubs & examples
      - base.py
      - examples.py
  - utils/
    - persistence.py      # save/load helpers
- tests/
  - unit/
    - test_board.py
    - test_cycle_detection.py
    - test_resolvers.py
  - integration/
    - test_playthroughs.py

### Core classes / responsibilities

* GameEngine: public API for making moves, applying resolver, checking game status. Emits events/hooks for UI and tests.
* Board: stores square objects, quantum links, classical assignments, serialization.
* QuantumMove: stores move id, player, endpoints, and any metadata.
* CollapseResolver (abstract interface): resolve(board, cycle, last_move) → Yields collapse options or directly applies collapse. Implementations:
  * OpponentChoiceResolver — returns options for UI; engine waits for opponent's selected collapse move.
  * ObjectiveResolver — deterministically computes collapse choice and applies it immediately.
* AnimationPlugin (OOP interface in animations/base.py): methods prepare(board, cycle), play(), finish(), teardown(); concrete example classes in animations/examples.py.

---

## UI behavior & controls

* Initial screen: 3x3 board, player display, toggle control "Collapse mode: [Opponent-choice | Objective]" with the AJP description shown on hover.
* Making a quantum move: click two different squares (or click square A then square B). Move is shown as a labeled link (e.g., "X1" between squares).
* When cycle is created:
  * In Opponent-choice mode: UI highlights cycle and prompts the opponent with collapse options. Opponent selects collapse option; engine applies and updates board.
  * In Objective mode: engine resolves immediately and plays collapse animation.
* Animation plugin: UI loads a default SimpleFade plugin. Plugins are optional, pluggable, and may exist in the repo as un-instantiated classes for later use.
* Toggle restrictions: collapse mode can only be set before the first move of a game. If toggled mid-game, show warning and require new game to switch.

---

## Determinism & randomness

* All engine-determined behavior that could affect tests must be deterministic by default. If randomness is used in ObjectiveResolver, provide a seedable RNG accessible from GameEngine so tests can reproduce outcomes.

---

## Tests & red/green/refactor discipline

Principle: every new behavior must be accompanied by tests. Small commits: each commit adds either a tiny feature AND its tests, a test-only commit, or a refactor commit (isolated and described).

Required tests:
* test_board.py — placement of quantum moves, serialization, and invariants.
* test_cycle_detection.py — various graphs of links to confirm cycle detection works.
* test_resolvers.py — tests for both OpponentChoiceResolver (ensures UI options are correct) and ObjectiveResolver (deterministic outcomes for given seeds).
* integration/test_playthroughs.py — canned sequences reproducing interesting game scenarios (AJP example sequences, ties, wins).

CI: include a GitHub Actions placeholder that runs pytest on push/PR.

---

## Commit & branching policy (enforced in team practice)

* Commit size: keep each commit small and focused. Prefer many tiny commits over one large commit except for refactors.
* Message format: <type>: <short description>
  * feat: new feature
  * fix: bugfix
  * test: tests added/changed
  * chore: housekeeping
  * refactor: large refactor (single commit per refactor)
* Refactor commits: must be single, self-contained commits with message refactor: <reason> and a short block in the body listing files changed and why.
* Branches: main (stable release), dev (daily), feature branches feat/<topic>.

---

## Logging of prompts

* Store the exact prompts issued by the user (you) to logs/prompts.log. Each line entry: ISO8601 timestamp, tab, the prompt text.
* Only append prompts; do not log assistant responses.
* The logs/ folder should be added to the repository and committed.

---

## Initial planned commits (small, sequential)

1. feat: add SPEC.md and conformance checklist — this file. (current)
2. chore: repo skeleton, README, CHANGELOG, logs/prompts.log — empty prompt log and folder structure.
3. feat: core engine models (Board, QuantumMove, GameEngine) and basic tests — unit tests for placements and serialization.
4. feat: cycle detection and tests — implement cycle-detection logic.
5. feat: resolvers (OpponentChoiceResolver + ObjectiveResolver) and tests — include deterministic ObjectiveResolver.
6. feat: minimal FastAPI server + static UI (toggle, click-to-place) — basic playable UI.
7. test: integration playthroughs — add canonical test sequences.
8. feat: animation plugin interface + simple plugin — OOP animations plugin system.
9. Further small enhancements: AI player, accessibility improvements, richer animations, CI workflow.

Each step should be a single commit (or a small set of commits with tests), not a single large commit bundling many steps. Refactors later should be single refactor: commits.

---

## Developer notes & constraints

* Keep code object-oriented to allow animation plugin drop-in.
* Keep UI minimal initially — focus on clear, testable mechanics.
* Maintain deterministic testable behavior; use seedable RNG when randomness is unavoidable.
* Do not change collapse mode mid-game.
* Keep prompt log handling minimal and robust (atomic append operations recommended).

---

## Open questions (to be resolved in early iterations)

1. Exact collapse algorithm for ObjectiveResolver — I will implement a deterministic rule that resolves cycles according to a reproducible heuristic (detailed in resolvers.py) and make it configurable; the tests will encode the canonical behavior. If you prefer a specific variant (e.g., minimize X wins, maximize ties, etc.), state it and tests will be adjusted.
2. UI tech choice: FastAPI + vanilla JS is the default. If you prefer pure client-side (single JS file), say so before the next commit.
3. Level of AI sophistication: minimal random/heuristic player by default. If you want stronger AI (minimax on classical collapsed state), specify.

---

## Next actions (I will perform now)

* Create the repo skeleton and README.md, CHANGELOG.md, and empty logs/prompts.log as described. This will be the first repo commit after SPEC.md.

---

End of SPEC.md
