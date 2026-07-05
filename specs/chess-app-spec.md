# Granpy Piero's AI Chess — Specification

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
- **G2.** Let the user independently assign White and Black to Human, Normal AI (with a difficulty), Grandmaster AI, or LLM-Assisted (with its own endpoint/model config).
- **G3.** Keep the UI responsive at all times, including during deep AI search.
- **G4.** Offer a genuinely wide strength range: from a beatable beginner AI up to super-human (Stockfish-class) play.
- **G5.** Keep the codebase understandable and self-contained enough that the Normal engine's logic is inspectable, not a black box.

## 3. Non-goals (out of scope for v1)

- **NG1.** Online multiplayer / networked play between two browsers.
- **NG2.** User accounts, persistent rating, or *automatic* game history across sessions. (Does not bar user-initiated PGN export/import — FR-8.2 — which is the user's explicit file choice, not silent persistence.)
- **NG3.** Inlining Stockfish into the single HTML file itself (as base64/inlined WASM). The Stockfish runtime ships as **sibling files committed to the repo** (NFR-5.2), not inlined into `chess.html` — keeps the HTML editable and small.
- **NG4.** Mobile-native app packaging (the deliverable is a web page; responsive layout is in scope, native app is not).
- **NG5.** Voice control, puzzles, tutorials, or opening-explorer features.

## 4. Users & match configuration

There is a single user-facing configuration surface: **for each color (White, Black), select a controller.**

| Controller | Description |
|---|---|
| Human | Moves are entered via the board UI (click or drag) |
| Normal AI | Local custom engine; requires a **Difficulty** sub-selection |
| Grandmaster AI | Stockfish WASM; no difficulty selection (max strength) |
| LLM-Assisted | Calls a user-configured OpenAI-compatible chat endpoint per move; requires **Endpoint / API key (optional) / Model** sub-selection |

All Human/AI combinations must be supported without special-casing in the rules engine (LLM-Assisted counts as an AI tier):

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
- FR-6.5 Destructive actions are confirmed. New Game, Rematch, and Resign
  during an in-progress game prompt a native `confirm()`; if the action would
  discard an in-progress game with moves, OK saves it as a PGN file first
  (FR-8.2) then proceeds, Cancel keeps playing. Resign only confirms (it
  completes the game, which can then be saved from the summary).

### FR-7. Game status & notation
- FR-7.1 Persistent status line reflecting game state: whose turn, check, checkmate, stalemate, draw (with reason).
- FR-7.2 On game end, display the result (1-0, 0-1, ½-½) and reason.

### FR-8. Persistence
- FR-8.1 No *automatic* persistence of game state in v1: refreshing the page
  resets to the setup screen. (Explicitly listed here, not left implicit, since
  "save game" is a common assumption to challenge in the PRD.) This bars
  automatic cross-session game history/ratings/accounts (NG2) **only**; it does
  not bar user-initiated file export/import (FR-8.2), which is the user's
  explicit choice and stays out of browser storage.
- FR-8.2 **PGN export/import (user-initiated).** At any point the user may
  download the current game as a `.pgn` file (movetext + tags, including
  `White`/`Black` controller labels and `Result`), and may load a `.pgn` file
  to replay it in a **view-only mode**: the board is reconstructed from the
  standard start position by matching each SAN token against the legal moves
  of the rebuilt position (no separate SAN parser — `toSAN` is reused as the
  matcher, with `+`/`#` stripped), states are precomputed once on load, and
  First/Prev/Next/Last controls step through plies. Loaded games do not
  resume engine/AI play (controllers are not carried by PGN); start a new game
  to play again. Custom-position games via a `[FEN]` tag are not supported in
  v1 (standard start only).

### FR-9. AI — LLM-Assisted mode
- FR-9.1 On selecting LLM-Assisted for a side, the user supplies, per side: an **API base URL** (OpenAI-compatible, stored as `apiBase`), an optional **API key** (blank works for local servers like Ollama / LM Studio), and a **Model** name. Both `apiBase` and `model` are required to start. The two sides may use different endpoints/models.
- FR-9.2 Each move is requested via `POST {apiBase}/chat/completions` (Authorization: Bearer {apiKey} when a key is supplied), sending the current FEN, the side to move, recent SAN move history, and the full list of legal UCI moves, and asking the model to return exactly one move from that list (constrained choice — the main reliability lever for legal LLM chess).
- FR-9.3 The reply is parsed for one UCI move (preferring an exact legal-move token) and **re-validated through the rules engine** like every other AI move (NFR-7.2). An illegal/unparseable reply is retried **up to once** with a corrective message that feeds the bad reply back into the conversation. If it still fails, or the network call itself fails, the side **falls back to a quick local Normal-engine move (difficulty 3) so play continues**, with a toast — LLM APIs fail transiently far more often than a local engine, and per the primary goal (entertain) a flaky API must not kill the game. Only if the fallback itself errors does the game end.
- FR-9.4 Setup **config** (per-side controller, difficulty, and LLM `apiBase`/`apiKey`/`model`) is **persisted to `localStorage` and proposed as defaults on the next restart** on the same computer. This is the single permitted persistence exception: it covers setup selections only, **not** game state/history/ratings (those remain barred by NG2 / FR-8.1). Persistence is best-effort and silently no-ops when storage is unavailable (e.g. disabled, or some `file://` contexts). Note the key is stored locally and readable by anyone with browser-profile access — acceptable for a single-user local tool, not for sharing the file with a real key typed in.
- FR-9.5 The eval bar and move-quality tags (PRD §5.4) remain functional in LLM-Assisted mode: the Normal engine exposes an `analyze` message (spec §9) that grades *any* played move — LLM or human — against a quick local search, returning `evalCp` (the played move's eval) and `bestEvalCp` (the best line's eval). The chat endpoint returns no eval, so the local engine acts purely as a judge, never as a mover — giving one yardstick to compare LLM moves against Normal and Grandmaster.

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

### NFR-5. Network dependency (Grandmaster and LLM-Assisted modes)
- NFR-5.1 Human vs Human and Human vs Normal AI must work fully offline once the page is loaded.
- NFR-5.2 Grandmaster runs **fully offline and out of the box**: the two-file Stockfish asset bundle (`stockfish-18-lite-single.js` + `stockfish-18-lite-single.wasm`) is **committed to the repo** next to `chess.html` (GPL-3.0 — §12.4). The loader tries the local bundle first, then falls back to a CDN; the CDN is currently non-functional anyway (jsDelivr refuses the >150 MB npm package with HTTP 403), but the committed bundle makes that irrelevant.
- NFR-5.4 LLM-Assisted mode requires network access to a user-supplied OpenAI-compatible endpoint on every move. The endpoint must itself serve permissive CORS headers (the browser makes the call directly); local servers such as LM Studio, llama.cpp, vLLM, and LocalAI do, while OpenAI's hosted API does not from a browser — a CORS-friendly proxy is the user's responsibility. No CORS handling is implemented app-side.
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
- **MatchConfig**: `{ white: ControllerConfig, black: ControllerConfig }` where `ControllerConfig` is `{ type: 'human' | 'normal' | 'grandmaster' | 'llm', difficulty?: 1-5, apiBase?: string, apiKey?: string, model?: string }` (the LLM fields apply iff `type === 'llm'`).

## 9. Cross-thread communication protocol (spec-level)

Both workers must accept/emit a normalized message shape so the main thread's game loop is engine-agnostic:

- **Main → Worker:** `{ type: 'search', fen, startFen, uciMoves, budget }` (plus `{ type: 'stop' }` to abort an in-flight search, for FR-6.4). The Normal engine also accepts `{ type: 'analyze', fen, uciMove }` to grade an arbitrary played move for PRD §5.4 quality tagging (used for LLM and human moves).
- **Worker → Main:** `{ type: 'result', move, evalCp?, bestEvalCp? }`, `{ type: 'analysis', evalCp, bestEvalCp }` (in reply to `analyze`), or `{ type: 'error', message }`

Exact message schema, versioning, and error codes are defined in the TDD.

## 10. Acceptance criteria (high-level — expand into test cases in TDD)

- AC-1: A full legal game can be played Human vs Human start to finish, correctly detecting checkmate/stalemate/draws.
- AC-2: Each Normal difficulty level is measurably different in strength/behavior (not just in name).
- AC-3: Grandmaster mode produces legal, strong moves via Stockfish WASM without freezing the UI.
- AC-4: All four controller combinations (Human/Human, Human/AI, AI/Human, AI/AI) can be configured and played.
- AC-5: The UI remains interactive during AI thinking at every difficulty and both tiers.
- AC-6: The app loads and is fully playable (Human vs Human, Human vs Normal AI) with no network connection.
- AC-7: A Stockfish load failure is handled gracefully and does not break the rest of the app.
- AC-8: LLM-Assisted mode produces legal moves from a configured OpenAI-compatible endpoint; an endpoint error or illegal reply (after retry) triggers a local fallback move rather than ending the game, and only a fallback failure freezes the UI gracefully.

## 11. Assumptions & open questions (to be resolved in PRD/TDD)

- A11.1 Exact ELO targets per Normal difficulty level — needs tuning/benchmarking, not specified here.
- A11.2 Whether Grandmaster mode exposes a "thinking time" or strength setting to the user, or is fixed — currently spec'd as fixed max strength.
- A11.3 ~~Whether multi-threaded WASM (requiring specific server headers) is a hard requirement or a "best effort if headers present"~~ — **Resolved (see §13):** use a single-threaded Stockfish WASM build for v1. Recommended package: `nmrugg/stockfish.js` single-threaded flavor (`stockfish` on npm). Multi-threading is a v1.1 upgrade gated on confirming the deployment host can serve COOP/COEP headers. See §13 and TDD §5.2.
- A11.4 Drag-and-drop move input is explicitly deferred to "nice to have" — confirm before PRD locks UX flows.
- A11.5 Sound effects, animations, and visual theme are UX decisions left entirely to the PRD.
- A11.6 LLM-Assisted prompt robustness — the single-hard-move prompt + one-retry policy (FR-9.3) is a baseline; real-world move legality/parse rates depend on the chosen model and may need a richer prompt or constrained decoding later. Not blocking v1.

## 12. Dependencies, references & licensing

This section records the third-party options surveyed for this project and the decisions taken. It is the authoritative dependency policy; the TDD (§5, §13) binds concrete packages to it.

### 12.1 Grandmaster engine — Stockfish WASM (used directly)
Surveyed ready-to-use in-browser Stockfish builds:

| Build | Strength / size | Threading / headers | License | Status |
|---|---|---|---|---|
| **`nmrugg/stockfish.js`** (`stockfish@18.0.0`) — single-threaded flavor | Full NNUE eval, single core | None required (no SharedArrayBuffer) | GPL-3.0 | **Chosen for v1** (resolves A11.3) |
| `nmrugg/stockfish.js` — multi-threaded flavor | Full NNUE, multi-core | Requires COOP/COEP (`SharedArrayBuffer`) | GPL-3.0 | v1.1 candidate, header-gated |
| `nmrugg/stockfish.js` — lite (~7MB) | Weaker, no full NNUE | Single-threaded, none | GPL-3.0 | Fallback if bundle size matters |
| `lichess-org/stockfish.wasm` | Classical eval, no NNUE, no Syzygy | Multi-threaded, requires COOP/COEP | GPL-3.0 | Not chosen (older eval, header-bound) |
| `lichess-org/stockfish.js` | Classical eval, ~250KB gzipped | Single-threaded, none | GPL-3.0 | Not chosen (smallest, but non-NNUE) |
| `hi-ogawa/Stockfish` | NNUE WASM port | Varies | Check in-repo | Reference only |

**Decision:** `nmrugg/stockfish.js` single-threaded. It carries full NNUE evaluation (so strength is unbounded relative to the Normal engine and well beyond human-Kasparov level) while running without COOP/COEP — directly satisfying NFR-5.3's "single-file app must work via `file://` or any static host" constraint. NPS is capped to one core, an accepted trade-off (TDD §5.2).

### 12.2 Rules engine — hand-rolled, with a de-risk fallback
The spec requires a self-authored rules engine (FR-1) so the Normal engine owns its own move generation and the codebase stays inspectable (G5). Surveyed libraries:

| Library | Lang | Coverage of FR-1 | License | Role here |
|---|---|---|---|---|
| **chess.js** | JS/TS | Full (FR-1 minus AI) | Check in-repo (historically permissive) | **Fallback**: drop in via CDN to de-risk FR-1 correctness if hand-rolled perft tests fail or schedule slips |
| **chessops** | TS | Full + variants | GPL-3.0 | Reference only |
| **shakmaty** | Rust | Full, bitboard/Zobrist, Lichess backend | GPL-3.0 | **Design reference** for bitboards, Zobrist hashing, FEN/SAN/UCI — study even though we implement 0x88 in JS (TDD §2, §3) |

**Decision:** implement the 0x88 rules engine per FR-1 (TDD §3); keep `chess.js` as the documented fallback, and study `shakmaty`'s design rather than reinvent it. This keeps FR-1 ownership while not betting correctness on a hand-rolled first pass.

### 12.3 Normal engine — design references
The Normal engine is authored in-house (FR-3). Reference implementations to study, not depend on:

| Project | Lang | Why study it |
|---|---|---|
| **Sunfish** | Python, ~111 lines | Canonical small engine; MTD-bi search, piece-square tables, null-move pruning; plays >2000 Lichess. Its README's "how to make it stronger" list maps ~1:1 onto TDD §4.3's difficulty ladder. Read end-to-end. |
| **Pleco** | Rust | Deliberate Stockfish-algorithm port; bitboard board rep + move gen under MIT (library crate). Reference for data structures. |
| **tmttn/chess** | Rust | Crate layout (core / engine / wasm-bindings) — reference if the Normal tier ever moves to WASM for speed (TDD §9). |

### 12.4 License policy
The project's own source and the Stockfish runtime it vendors are both **GPL-3.0**, so the whole repository is GPL-3.0 — consistent copyleft across the app and the bundled engine.

- **The project is licensed under the GNU GPL v3.0** (`LICENSE` at the repo root).
- **The Stockfish runtime is vendored into the repo** as two sibling files (`stockfish-18-lite-single.{js,wasm}`) next to `chess.html`, so Grandmaster works out of the box — offline and on the live site. This is a deliberate licensing choice: because Stockfish (GPL-3.0) is a derivative/combined-work component, the project embraces GPL-3.0 copyleft rather than trying to keep it at arm's length.
- Most other strong chess libs (`shakmaty`, `chessops`) are also GPL-3.0 but remain **design references only** (the rules engine is hand-rolled 0x88 — FR-1, no GPL `chessops`/`shakmaty` linkage). They could be vendored later without further license impact; today they are not.
- The prior (Apache-2.0) posture kept Stockfish as a UCI-over-worker runtime dependency and forbade vendoring it; that constraint is **lifted** by this relicense.

### 12.5 Summary decisions
| Concern | Decision |
|---|---|
| Grandmaster engine | `nmrugg/stockfish.js`, single-threaded flavor, loaded from CDN via worker `importScripts` |
| Rules engine | Hand-rolled 0x88 (FR-1); `chess.js` fallback; `shakmaty` as design reference |
| Normal engine | In-house (FR-3); Sunfish/Pleco as study references |
| License posture | Whole project GPL-3.0; Stockfish runtime vendored into the repo (`stockfish-18-lite-single.{js,wasm}`) so Grandmaster works out of the box |

## 13. Glossary

- **SDD** — Spec-Driven Development.
- **PRD** — Product Requirements Document (derived from this spec; defines user-facing flows and UX).
- **TDD** — Technical Design Document (derived from this spec; defines architecture, modules, and data structures).
- **FEN** — Forsyth–Edwards Notation, a compact text representation of a chess position.
- **SAN** — Standard Algebraic Notation, e.g. `Nf3`, `O-O`, `exd5`.
- **UCI** — Universal Chess Interface, the text protocol Stockfish and most engines speak.
- **NNUE** — Efficiently updatable neural network, Stockfish's modern evaluation approach.
