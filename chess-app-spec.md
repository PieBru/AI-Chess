# Single-File AI Chess — Specification

**Document type:** Spec (SDD Layer 1 — WHAT and WHY)
**Downstream artifacts:** PRD (product/UX detail), TDD (technical/architecture detail)
**Status:** Draft v1.0

---

## 0. Methodology note

This document is the top-level specification in a Spec-Driven Development flow:

```
SPEC (this doc)  →  PRD                    →  TDD
"what must be     "user-facing behavior,     "how it's built: modules,
true, and why"     flows, copy, UX detail"    data structures, algorithms,
                                               worker protocol, file layout"
```

This spec defines scope, requirements, and constraints at a level independent of implementation detail or UI phrasing. It is the source of truth both derived documents must trace back to — every PRD user flow and every TDD component should map to a requirement ID below (e.g. `FR-3.2`).

---

## 1. Overview

A chess application delivered as a single self-contained HTML file (inline CSS + JavaScript) playable in any modern browser with no build step, no backend, and no installation. Any combination of Human and AI can occupy either side of the board: Human vs Human, Human vs AI, AI vs Human, or AI vs AI.

Two independent AI tiers are offered:

- **Normal** — a hand-written chess engine (classical search: alpha-beta, iterative deepening, quiescence search, evaluation function) with selectable difficulty levels, computed entirely locally.
- **Grandmaster** — Stockfish compiled to WebAssembly, offering near-maximum engine strength.

Both engines run inside a **Web Worker** so the UI thread never blocks during search.

## 2. Goals

- **G1.** Provide a fully playable, rules-correct chess game in one HTML file.
- **G2.** Let the user independently assign White and Black to Human, Normal AI (with a difficulty), or Grandmaster AI.
- **G3.** Keep the UI responsive at all times, including during deep AI search.
- **G4.** Offer a genuinely wide strength range: from a beatable beginner AI up to super-human (Stockfish-class) play.
- **G5.** Keep the codebase understandable and self-contained enough that the Normal engine's logic is inspectable, not a black box.

## 3. Non-goals (out of scope for v1)

- **NG1.** Online multiplayer / networked play between two browsers.
- **NG2.** User accounts, persistent rating, or game history across sessions.
- **NG3.** Full offline operation in Grandmaster mode (see §6.5 — Stockfish WASM is expected to load from a CDN; a fully bundled offline build is a future consideration, not a v1 requirement).
- **NG4.** Mobile-native app packaging (the deliverable is a web page; responsive layout is in scope, native app is not).
- **NG5.** Voice control, puzzles, tutorials, or opening-explorer features.

## 4. Users & match configuration

There is a single user-facing configuration surface: **for each color (White, Black), select a controller.**

| Controller | Description |
|---|---|
| Human | Moves are entered via the board UI (click or drag) |
| Normal AI | Local custom engine; requires a **Difficulty** sub-selection |
| Grandmaster AI | Stockfish WASM; no difficulty selection (max strength) |

All four combinations must be supported without special-casing in the rules engine:

- Human vs Human
- Human vs AI (either tier, either color)
- AI vs AI (same or different tiers) — the app must be able to play itself out, useful for demos/testing

## 5. Functional requirements

### FR-1. Chess rules engine
- FR-1.1 Legal move generation for all pieces, including castling (kingside/queenside, with all legality conditions: king/rook unmoved, no intervening pieces, king not in/through/into check), en passant, and pawn promotion (must prompt for piece choice when a human promotes; AI selects programmatically).
- FR-1.2 Check, checkmate, and stalemate detection.
- FR-1.3 Draw detection: threefold repetition, fifty-move rule, insufficient material.
- FR-1.4 A move is only committed to the game state if fully legal; illegal move attempts by a human are rejected with visual feedback and do not change state.
- FR-1.5 The engine must produce a FEN string for any position and standard algebraic notation (SAN) for any move, since both AI tiers and the move-history UI depend on them.

### FR-2. Game modes & player assignment
- FR-2.1 Before a game starts, the user selects White's controller and Black's controller independently, per §4.
- FR-2.2 If Normal AI is selected for a side, a Difficulty must also be selected (see FR-3.2).
- FR-2.3 Controller assignment can be changed via "New Game"; it is not changeable mid-game.
- FR-2.4 When it is an AI-controlled side's turn, the UI must clearly indicate "thinking" state and disable human move input on the board.

### FR-3. AI — Normal mode
- FR-3.1 Search algorithm requirements: negamax with alpha-beta pruning, iterative deepening with a time or depth budget, transposition table (Zobrist hashing), quiescence search at leaf nodes, and move ordering (transposition-table move, MVV-LVA for captures, killer moves, history heuristic).
- FR-3.2 Difficulty levels — minimum five, tuned by search depth/time and move-selection noise:

  | Level | Approx. depth/time budget | Move selection | Target feel |
  |---|---|---|---|
  | 1 — Beginner | ~1 ply / ~50ms | Weighted random among (wide) legal moves, biased toward reasonable; occasional intentional non-top move | Loses to a novice |
  | 2 — Easy | ~2–3 ply / ~150ms | Random among top-3 moves | Casual player |
  | 3 — Medium | ~4 ply / ~500ms | Best move, small eval noise | Club player |
  | 4 — Hard | ~6 ply / ~1.5s | Best move, minimal noise | Strong club player |
  | 5 — Expert | full iterative deepening / ~3–5s budget | Best move only | Upper limit of this engine's honest strength |

  Exact ELO targets are a PRD/TDD tuning concern; this table constrains *mechanism* (how difficulty is achieved), not exact numbers.
- FR-3.3 The Normal engine must never crash or stall the worker on any legal position, including positions with very few legal moves (forced sequences) or none (must correctly report game-over rather than search).
- FR-3.4 Evaluation function must include, at minimum: material balance, piece-square tables (tapered between middlegame/endgame), mobility, and basic king safety.

### FR-4. AI — Grandmaster mode
- FR-4.1 Uses Stockfish compiled to WebAssembly, communicated with via the standard UCI text protocol.
- FR-4.2 Runs at effectively unrestricted strength (no artificial ELO cap) within a bounded thinking time per move, configurable as a constant (not user-facing in v1).
- FR-4.3 If the Stockfish WASM asset fails to load (e.g. offline, blocked CDN), the app must degrade gracefully: surface a clear error to the user and prevent selecting Grandmaster mode, rather than silently failing mid-game.
- FR-4.4 Grandmaster engine selection must not require any server-side component — asset loading is client-side only (script tag / fetch from CDN).

### FR-5. Board & interaction UI
- FR-5.1 Rendered 8×8 board, correctly oriented (can be flipped), with legal-move highlighting when a human selects a piece.
- FR-5.2 Move input via click-to-select-then-click-to-move at minimum; drag-and-drop is a nice-to-have, not required for v1.
- FR-5.3 Last-move highlight, check indicator, and captured-piece tray for both sides.
- FR-5.4 Move history panel in SAN, scrollable, updated after every ply.

### FR-6. Game controls
- FR-6.1 New Game (re-opens controller/difficulty selection).
- FR-6.2 Resign (available to a human player, ends game with the opponent as winner).
- FR-6.3 Flip board.
- FR-6.4 Pause/stop AI-vs-AI games (since these run unattended).

### FR-7. Game status & notation
- FR-7.1 Persistent status line reflecting game state: whose turn, check, checkmate, stalemate, draw (with reason).
- FR-7.2 On game end, display the result (1-0, 0-1, ½-½) and reason.

### FR-8. Persistence
- FR-8.1 No persistence requirement in v1: refreshing the page resets to the setup screen. (Explicitly listed here, not left implicit, since "save game" is a common assumption to challenge in the PRD.)

## 6. Non-functional requirements

### NFR-1. Single-file architecture
The deliverable is one `.html` file containing all CSS and JS inline. Exception: the Stockfish WASM engine and its network weight file may be loaded from an external CDN at runtime (see NFR-5) — this is the one permitted external dependency, since embedding a multi-megabyte WASM binary as base64 is impractical and against the spirit of an editable single-file app.

### NFR-2. Performance
- NFR-2.1 UI interactions (piece selection, highlighting, menu navigation) must respond within 100ms regardless of AI activity.
- NFR-2.2 Normal AI must respect its difficulty's time/depth budget (§FR-3.2) and not exceed it by more than a small margin.

### NFR-3. Responsiveness (non-blocking UI)
- NFR-3.1 Both AI tiers must run their search inside a Web Worker. The main thread must never run a blocking search loop.
- NFR-3.2 The board and controls must remain interactive (e.g. flipping the board, scrolling move history) while an AI is thinking.

### NFR-4. Compatibility
- NFR-4.1 Must run in current versions of Chrome, Firefox, Safari, and Edge without polyfills for standard Web Worker / WebAssembly APIs.
- NFR-4.2 Must be usable on both desktop and mobile viewport widths (responsive layout).

### NFR-5. Network dependency (Grandmaster mode only)
- NFR-5.1 Human vs Human and Human vs Normal AI must work fully offline once the page is loaded.
- NFR-5.2 Grandmaster mode requires network access on first use (to fetch the Stockfish WASM asset). This is an explicit, documented exception to "single file" — see NG3.
- NFR-5.3 **Threading caveat to carry into the TDD:** multi-threaded Stockfish WASM builds require `SharedArrayBuffer`, which requires cross-origin isolation (COOP/COEP response headers). A file opened directly via `file://` or served without those headers will not get multi-threading. The TDD must decide between (a) using a single-threaded WASM build for maximum compatibility at somewhat reduced NPS, or (b) requiring the file be served with the correct headers to unlock multi-threading. This materially affects Grandmaster-mode strength and must not be left ambiguous downstream.

### NFR-6. Accessibility
- NFR-6.1 Board and controls must be keyboard-navigable at a basic level (tab to controls, at minimum).
- NFR-6.2 Sufficient color contrast for board squares, highlights, and status text.

### NFR-7. Reliability
- NFR-7.1 A worker crash or unhandled engine error must not freeze the page — it must be caught, surfaced to the user, and allow starting a new game.
- NFR-7.2 The rules engine must be defensive against corrupt/unreachable game states (e.g. no legal moves available) and never allow an illegal position to be reached.

## 7. AI engine specification

### 7.1 Shared architecture
Both AI tiers are accessed through a **common interface** from the main thread's perspective, regardless of which worker or protocol backs them:

```
requestMove(fen, gameHistory, config) → Promise<{ move, ponder?, evalCp?, bestEvalCp? }>
// evalCp/bestEvalCp are centipawns from the mover's perspective; TDD §6 names these canonically.
```

This abstraction is what allows FR-2's "any controller on any side" requirement to be implemented without special-casing — the game loop doesn't know or care whether it's talking to the Normal engine or Stockfish.

### 7.2 Normal engine
- Owns its own Web Worker, running the custom JS/WASM search (§FR-3).
- Receives: FEN, difficulty level, time/depth budget.
- Returns: chosen move (plus optionally a centipawn score, used by the eval-bar UI in PRD §5.4).

### 7.3 Grandmaster engine
- Owns a separate Web Worker wrapping the Stockfish WASM build, communicating via UCI text commands (`position fen ...`, `go movetime ...`, reading `bestmove ...` from output).
- Receives: FEN, move history (for repetition awareness), and a fixed internal think-time.
- Returns: chosen move parsed from the engine's `bestmove` output.

### 7.4 Engine lifecycle
- Engines are instantiated lazily — only when a game actually needs that tier — to avoid loading the Stockfish WASM asset for games that never use Grandmaster mode.
- Workers are torn down / terminated on "New Game" if the controller selection changes, to free memory.

## 8. Data model (spec-level — detailed schema is a TDD concern)

- **GameState**: board position, side to move, castling rights, en passant target, halfmove clock, fullmove number, move history (SAN + FEN per ply), result.
- **Move**: from-square, to-square, piece, capture flag, promotion piece (if any), special-move flag (castle/en passant).
- **MatchConfig**: `{ white: ControllerConfig, black: ControllerConfig }` where `ControllerConfig` is `{ type: 'human' | 'normal' | 'grandmaster', difficulty?: 1-5 }`.

## 9. Cross-thread communication protocol (spec-level)

Both workers must accept/emit a normalized message shape so the main thread's game loop is engine-agnostic:

- **Main → Worker:** `{ type: 'search', fen, startFen, uciMoves, budget }` (plus `{ type: 'stop' }` to abort an in-flight search, for FR-6.4)
- **Worker → Main:** `{ type: 'result', move, evalCp?, bestEvalCp? }` or `{ type: 'error', message }`

Exact message schema, versioning, and error codes are defined in the TDD.

## 10. Acceptance criteria (high-level — expand into test cases in TDD)

- AC-1: A full legal game can be played Human vs Human start to finish, correctly detecting checkmate/stalemate/draws.
- AC-2: Each Normal difficulty level is measurably different in strength/behavior (not just in name).
- AC-3: Grandmaster mode produces legal, strong moves via Stockfish WASM without freezing the UI.
- AC-4: All four controller combinations (Human/Human, Human/AI, AI/Human, AI/AI) can be configured and played.
- AC-5: The UI remains interactive during AI thinking at every difficulty and both tiers.
- AC-6: The app loads and is fully playable (Human vs Human, Human vs Normal AI) with no network connection.
- AC-7: A Stockfish load failure is handled gracefully and does not break the rest of the app.

## 11. Assumptions & open questions (to be resolved in PRD/TDD)

- A11.1 Exact ELO targets per Normal difficulty level — needs tuning/benchmarking, not specified here.
- A11.2 Whether Grandmaster mode exposes a "thinking time" or strength setting to the user, or is fixed — currently spec'd as fixed max strength.
- A11.3 Whether multi-threaded WASM (requiring specific server headers) is a hard requirement or a "best effort if headers present" — see NFR-5.3.
- A11.4 Drag-and-drop move input is explicitly deferred to "nice to have" — confirm before PRD locks UX flows.
- A11.5 Sound effects, animations, and visual theme are UX decisions left entirely to the PRD.

## 12. Glossary

- **SDD** — Spec-Driven Development.
- **PRD** — Product Requirements Document (derived from this spec; defines user-facing flows and UX).
- **TDD** — Technical Design Document (derived from this spec; defines architecture, modules, and data structures).
- **FEN** — Forsyth–Edwards Notation, a compact text representation of a chess position.
- **SAN** — Standard Algebraic Notation, e.g. `Nf3`, `O-O`, `exd5`.
- **UCI** — Universal Chess Interface, the text protocol Stockfish and most engines speak.
- **NNUE** — Efficiently updatable neural network, Stockfish's modern evaluation approach.
