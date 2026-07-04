# Single-File AI Chess — Technical Design Document (TDD)

**Document type:** TDD (SDD Layer 2 — architecture, data, algorithms)
**Derived from:** `chess-app-spec.md`, `chess-app-prd.md`
**Status:** Draft v1.0

---

## 1. Architecture overview

Single HTML file with four logical layers, all inline but organized as separate `<script>` sections or IIFE modules for readability. No layer is a separate file — this is an organizational convention within one document, not a module system.

```
┌─────────────────────────────────────────────┐
│ Main thread                                  │
│  ┌───────────┐  ┌────────────┐  ┌─────────┐ │
│  │ Rules      │  │ Game loop /│  │ UI /    │ │
│  │ engine     │◄─┤ state mgr  │─►│ render  │ │
│  │ (module)   │  └─────┬──────┘  └─────────┘ │
│  └───────────┘        │                      │
│                        ▼                      │
│              ┌──────────────────┐            │
│              │ Engine manager     │            │
│              │ (common interface) │            │
│              └──┬──────────────┬─┘            │
└─────────────────┼──────────────┼──────────────┘
                   ▼              ▼
        ┌──────────────────┐  ┌──────────────────┐
        │ Normal Worker      │  │ Grandmaster Worker │
        │ (custom JS engine) │  │ (Stockfish WASM)   │
        └──────────────────┘  └──────────────────┘
```

- **Rules engine** — pure functions, no DOM/worker dependency. Owns board representation, legal move generation, FEN/SAN, game-end detection.
- **Game loop / state manager** — owns the single source-of-truth `GameState`, decides whose turn it is and which controller acts, dispatches to the engine manager or waits for human input, applies the resulting move via the rules engine.
- **Engine manager** — implements the common `requestMove()` interface from the spec (§7.1), lazily instantiates whichever worker(s) are needed, normalizes both workers' outputs into one shape (including eval score, for PRD §5.4), and terminates/recreates workers when the controller selection changes on New Game (spec §7.4).
- **UI/render** — DOM-based board and panels; subscribes to state manager events, never mutates game state directly (all input funnels through the state manager, which validates via the rules engine).

## 2. Data structures

```ts
// Board: 0x88 mailbox representation — simple, debuggable, fast enough
// for a browser engine at these depths. 128-length Int8Array; valid
// squares are indices where (i & 0x88) === 0.
type Board = Int8Array; // piece codes: 0 = empty, ±1..±6 = pawn..king, sign = color

interface GameState {
  board: Board;
  sideToMove: 'w' | 'b';
  castlingRights: { wK: bool; wQ: bool; bK: bool; bQ: bool };
  epTarget: number | null;      // 0x88 square index or null
  halfmoveClock: number;        // for 50-move rule (FR-1.3)
  fullmoveNumber: number;
  history: MoveRecord[];        // full game so far
  result: 'in_progress' | '1-0' | '0-1' | '1/2-1/2';   // internal ASCII enum; rendered as `½-½` for display (spec FR-7.2)
  resultReason: string | null;
}

interface MoveRecord {
  move: Move;
  san: string;
  fenAfter: string;
  evalCp: number | null;        // centipawns, from the mover's engine if AI; null for human
  bestEvalCp: number | null;    // best available move's eval, for quality tagging (PRD §5.4)
  quality: 'best'|'good'|'inaccuracy'|'mistake'|'blunder'|null;
}

interface Move {
  from: number; to: number;     // 0x88 indices
  piece: number;
  capture: boolean;
  promotion: number | null;     // piece code or null
  flag: 'normal'|'castle_k'|'castle_q'|'en_passant'|'double_push';
}

interface ControllerConfig {
  type: 'human' | 'normal' | 'grandmaster' | 'llm';
  difficulty?: 1|2|3|4|5;       // required iff type === 'normal'
  endpoint?: string;            // required iff type === 'llm' (OpenAI-compatible base URL)
  apiKey?: string;              // optional iff type === 'llm'
  model?: string;               // required iff type === 'llm'
}

interface MatchConfig {
  white: ControllerConfig;
  black: ControllerConfig;
  spectatorSpeed?: 'normal'|'fast'|'instant'; // PRD §2.3, AI-vs-AI only
}
```

Zobrist hashing table (for the Normal engine's transposition table) is generated once at startup: a fixed-seed PRNG populates a `[piece][square][color]` random 32/53-bit key table plus keys for castling rights, en passant file, and side to move, XORed together to produce each position's hash.

## 3. Rules engine design

> **Design references (study, don't depend on):** `shakmaty` (Rust, Lichess backend) for bitboard + Zobrist + FEN/SAN/UCI correctness patterns; **`chess.js`** is the documented fallback (spec §12.2) — if perft tests (§11) fail or FR-1 schedule slips, drop it in via CDN rather than ship a buggy rules core. We implement the 0x88 approach below in-house to keep FR-1 owned and the engine inspectable (spec G5).

- **Move generation**: pseudo-legal generation per piece type using precomputed offset tables (knight/king jump deltas, sliding-piece direction deltas for bishop/rook/queen), then a legality filter that simulates each move and checks whether the mover's king is attacked afterward.
- **Attack detection**: a single `isSquareAttacked(board, square, byColor)` function reused for legality filtering, check detection, and castling-through-check validation — one implementation, multiple call sites, to avoid divergent logic (a common source of chess-engine rules bugs).
- **Game-end detection**: after generating legal moves for the side to move, zero legal moves + king in check = checkmate; zero legal moves + king not in check = stalemate. Threefold repetition tracked via a `Map<fenPositionKey, count>` keyed on the position-relevant subset of FEN (board + side + castling + ep, excluding clocks). Insufficient material checked via a simple material-count table (K vs K, K+B vs K, K+N vs K, K+B vs K+B same-color bishops).
- **FEN/SAN**: FEN generation is a direct serialization of `GameState`. SAN generation requires disambiguation logic (checking whether other same-type pieces could also reach the destination square), promotion suffixes (`=Q`/`=R`/`=B`/`=N`), castling notation (`O-O`/`O-O-O`), and `+`/`#` check/checkmate suffixes — implemented once, used both for the move-history UI and internally if useful for debugging.

## 4. Normal engine design (custom JS, Web Worker)

> **Primary design reference: Sunfish** (Python, ~111 lines). Its MTD-bi search loop, piece-square-table eval, and null-move pruning are close in spirit to §4.1–§4.2, and its README's "how to make it stronger" list maps almost 1:1 onto the §4.3 difficulty ladder (mutable board → bitboards, dedicated capture gen + check detection, MVV-LVA + SEE move ordering). Also reference **Pleco** (Rust, Stockfish-algorithm port; MIT library crate) for bitboard data-structure patterns. Neither is a dependency — the engine is authored in-house (spec FR-3).

### 4.1 Search
- **Algorithm**: negamax with alpha-beta pruning.
- **Iterative deepening**: loop from depth 1 upward until the difficulty's time or depth budget (table below) is exhausted; always keep the last *completed* depth's best move (never return a partial/interrupted search result).
- **Transposition table**: `Map` (or a fixed-size typed-array hash table for higher difficulties, to bound memory and avoid GC pauses) keyed by Zobrist hash, storing depth, score, node type (exact/lower/upper bound), and best move.
- **Move ordering**: TT move first, then captures sorted by MVV-LVA, then two killer-move slots per ply, then history heuristic for remaining quiet moves.
- **Quiescence search**: at depth 0, continue searching captures (and, at low extra depth, checks) until the position is quiet, to avoid the horizon effect on tactical sequences.
- **Pruning/reductions** (difficulty-gated, see §4.3): null-move pruning and late-move reductions enabled from Difficulty 3 upward; omitted at 1–2 where search is already shallow enough that the added complexity isn't worth the risk of pruning away a "human-like" blunder-punishing line (part of what makes low difficulties beatable rather than just "shallow but still tactically sharp").

### 4.2 Evaluation function
Tapered evaluation, blended by a game-phase counter (sum of remaining non-pawn material relative to starting material):
- Material (standard values, tunable constants).
- Piece-square tables per piece type, separate middlegame/endgame tables, linearly blended by phase.
- Mobility: count of legal (or pseudo-legal, for speed) moves per side, small weight.
- King safety: pawn shield presence in front of a castled king, penalty for open files near the king.
- Bishop pair bonus.

### 4.3 Difficulty parameter table (implements spec FR-3.2 / PRD §5.1)

| Level | Max depth | Time budget | Pruning enabled | Move selection |
|---|---|---|---|---|
| 1 | 1 | 50ms | No | Weighted random among all legal moves, biased toward reasonable ones; small chance of a randomly-selected non-top move even when a clear best move exists |
| 2 | 3 | 150ms | No | Random among top 3 by eval |
| 3 | 4 | 500ms | Yes | Best move + small Gaussian eval noise before comparison |
| 4 | 6 | 1500ms | Yes | Best move, minimal noise |
| 5 | ∞ (iterative) | 3000–5000ms | Yes (full) | Best move only |

Exact constants are implementation-tunable; this table is the contract the difficulty selector must honor.

### 4.4 Worker responsibilities
- Owns board state reconstruction from a FEN string per request (stateless between calls — simplest correctness story, avoids state-drift bugs between main thread and worker).
- Reports intermediate progress messages during iterative deepening (`{ type: 'progress', depth, evalCp }`) so the main thread can drive the PRD §5.3 thinking indicator without polling.
- Returns final `{ type: 'result', move, evalCp, bestEvalCp }` — `bestEvalCp` is the same as `evalCp` except at Difficulty 1–3 where move-selection noise/randomization may pick a move other than the top-scored one; the engine manager needs both values for the PRD §5.4 quality-tagging feature to work even at low difficulties.

## 5. Grandmaster engine design (Stockfish WASM, Web Worker)

### 5.1 Asset loading
- **Chosen build: `nmrugg/stockfish.js`, single-threaded flavor** (`npm i stockfish`, GPL-3.0) — full NNUE evaluation, single core, no `SharedArrayBuffer`/COOP/COEP requirement (spec §12.1, §12.4). This directly resolves the open question flagged in spec NFR-5.3 / A11.3.
- Loaded from a CDN via `importScripts()` inside the worker (or `fetch` + `WebAssembly.instantiate`, depending on the build's loader). Loaded lazily — only when a Grandmaster-controlled game actually starts (PRD/spec FR-4.4, NG3).
- **License posture:** the Stockfish worker is a separate runtime process speaking UCI over the post (spec §7.3), not statically linked into our source — keeps GPL-3.0 copyleft on the asset, off our JavaScript (spec §12.4).
- Load failure (network error, blocked resource) triggers a worker message `{ type: 'error', code: 'load_failed' }`; the engine manager surfaces this to the UI per PRD §5.5 and disables the Grandmaster option for the current session rather than retrying silently on every move. The single-threaded build removes the COOP/COEP failure mode entirely (the one deployment header class that previously could make Grandmaster silently degrade).

### 5.2 Threading decision (resolves spec open question A11.3 / NFR-5.3)
**Decision: use the single-threaded `nmrugg/stockfish.js` build for v1.**
Rationale: the single-file app must work when simply opened as a local file or served from an arbitrary static host without guaranteed COOP/COEP headers. A multi-threaded build (`SharedArrayBuffer`) silently degrades to single-threaded or fails outright in that common case. The single-threaded `nmrugg` build is designed for exactly this constraint (spec §12.1). Treat multi-threading as a v1.1 upgrade gated on confirming the deployment target can serve the required `Cross-Origin-Embedder-Policy`/`Cross-Origin-Opener-Policy` headers — at which point swapping in `nmrugg`'s multi-threaded flavor (or `lichess-org/stockfish.wasm`) is a worker-asset swap, not an architecture change.
Consequence: Grandmaster strength is bounded by single-thread NPS. This is still far beyond the Normal engine and beyond human-Kasparov-level play at reasonable think times (spec context), so it does not compromise the product goal — flagged here so it isn't silently forgotten.

### 5.3 UCI protocol wrapper
- On worker init: send `uci`, wait for `uciok`; send `isready`, wait for `readyok`.
- Per move request: send `position fen <startFen> moves <uciMoves>` (both supplied directly in the request, built by the engine manager from the rules-engine `Move` history so Stockfish carries full repetition context — required by spec §7.3; a bare current FEN would blind it to threefold-repetition eval/draws), then `go movetime <N>` (fixed internal constant, not user-exposed per PRD scope) or `go depth <N>` as a fallback if movetime-based timing proves inconsistent across browsers.
- Parse `info depth <d> score cp <cp> ...` lines during search to extract the live eval for the thinking indicator (PRD §5.3) and to obtain `bestEvalCp` for move-quality tagging (PRD §5.4) — Stockfish reports its own top-line eval as it deepens, so no extra search call is needed to get this number.
- Parse `bestmove <uci-move> [ponder <move>]` as the terminal response.
- UCI move strings (e.g. `e2e4`, `e7e8q`) are converted to the internal `Move` shape via the rules engine's own move generator (find the legal move matching that from/to/promotion) rather than trusted blindly — this reuses the legality filter as a safety net against any parser edge case.

### 5.4 LLM-Assisted engine (main-thread fetch, implements spec FR-9)
Unlike the Normal and Grandmaster engines, the LLM-Assisted controller is **not a worker** — it issues a direct `fetch()` from the main thread to the user-supplied OpenAI-compatible endpoint. It still satisfies the engine-manager `requestMove()` contract, returning the normalized `{ move, evalCp, bestEvalCp }` shape so the game loop stays agnostic (spec §7.1).

- **Request:** `POST {endpoint}/chat/completions` with `Authorization: Bearer {apiKey}` when a key is supplied; body is `{ model, messages, temperature: 0, max_tokens: 8 }`. The system message constrains the model to reply with exactly one legal UCI move; the user message supplies the FEN and the full legal UCI move list.
- **Parse:** extract the first `[a-h][1-8][a-h][1-8][qrbn]?` match from `choices[0].message.content`.
- **Legality:** the parsed move must appear in the legal list; an illegal/unparseable reply is retried **once** (same request, no state change), then rejected as an engine error (spec FR-9.3). The chosen move is re-validated through the rules engine on the game-loop side like every other AI move (§9, NFR-7.2).
- **Evals:** the chat endpoint returns no eval, so `bestEvalCp` is derived from a Normal-engine search of the current position (difficulty 4) and `evalCp` from a Normal-engine search of the child position after the LLM's move (negated) — reusing the existing worker, no new eval path. This keeps the eval bar (PRD §5.4) and move-quality tags working in LLM mode.
- **Failure handling:** a non-2xx response, network/CORS error, or exhausted retry is surfaced through the same `§5.5` worker-crash path (toast, freeze board, New Game) — the game loop's `try/catch` already covers it since `requestMove` rejects.
- **CORS / security (ponytail):** the endpoint must serve permissive CORS itself; no app-side proxy or CORS handling. API keys are held only in memory for the session (no `localStorage`) — re-entered per FR-9.4. This is acceptable for a local demo-grade tool; revisit if exposed publicly.

## 6. Worker communication protocol (concrete schema, implements spec §9)

```ts
// Main → Worker (both engine workers accept this shape)
type WorkerRequest =
  | { type: 'init' }
  | { type: 'search'; fen: string; startFen: string; uciMoves: string[]; budget: { depth?: number; timeMs?: number }; difficulty?: number }   // `fen` = current position (Normal engine uses it directly); `startFen`+`uciMoves` = full move line for Stockfish repetition context (spec §7.3)
  | { type: 'stop' };   // abort the current search (FR-6.4); Normal engine returns best-so-far, Grandmaster sends UCI `stop`

// Worker → Main
type WorkerResponse =
  | { type: 'ready' }
  | { type: 'progress'; depth: number; evalCp: number }
  | { type: 'result'; move: Move; evalCp: number; bestEvalCp: number }
  | { type: 'error'; code: string; message: string };
```

The Engine Manager is the only consumer of raw worker messages; it exposes `requestMove(fen, history, config): Promise<EngineResult>` to the game loop, hiding the two different underlying protocols (custom JSON vs UCI text parsed into this shape) behind one interface — this is what lets the game loop remain agnostic to which tier is playing (spec §7.1).

## 7. UI/render layer

- Plain DOM manipulation (no framework — keeps the single file dependency-free); board rendered as an 8×8 grid of `<div>`/`<button>` squares, pieces as inline SVG or a Unicode/webfont glyph set for compactness.
- State manager emits events (`move_made`, `turn_changed`, `thinking_started`, `thinking_progress`, `game_over`) that the UI layer subscribes to; UI never reaches into `GameState` directly except read-only for rendering, enforcing the one-way data flow described in §1.
- Eval bar (PRD §5.4) is a simple width/height-scaled div driven by the latest `evalCp` from state, clamped to a display range (e.g. ±1000cp) with a "mate in N" special case when a worker reports a mate score. Both engines report scores side-to-move-relative (negamax convention for Normal, UCI `score cp` for Stockfish), so the engine manager normalizes `evalCp` to White's perspective before storing it (negate when the mover was Black) — otherwise the bar's polarity would flip every ply. In a game with no AI mover (pure Human vs Human) there is no engine to produce `evalCp`; to satisfy PRD §5.4's "pending until an engine scores it" behavior, the engine manager spawns a Normal worker for a one-shot shallow background eval of the position after each human move (cheap, lazy, torn down when leaving H-vs-H).

## 8. State management & game loop

```
loop:
  if gameState.result !== 'in_progress': render summary, stop
  controller = matchConfig[gameState.sideToMove]
  if controller.type === 'human':
      wait for validated UI move → apply → continue loop
  else:
      emit thinking_started
      result = await engineManager.requestMove(fen, gameHistory, controller)
      apply result.move (validated through rules engine regardless of source)
      record evalCp / bestEvalCp / quality tag on the MoveRecord
      if spectatorSpeed set and both sides are AI: await pacing delay (PRD §2.3)
      continue loop
```

All applied moves — human or AI — pass through the same rules-engine validation function before touching `GameState`. This is a deliberate defensive measure: it means a bug in either AI worker can never corrupt the game (spec NFR-7.2), since an illegal move returned by a worker would simply be rejected and treated as a worker error.

**Pause / Stop (FR-6.4):** both act at the game-loop level on the next iteration boundary. *Pause* cancels the pending `spectatorSpeed` pacing delay and holds before requesting the next AI move (Resume clears it); *Stop* sends `{type:'stop'}` to any in-flight worker search and ends the game. An already-in-flight search is bounded by its own time/depth budget (§4.3 / §5.3), so worst case the current move completes within budget before the loop halts — no unbounded blocking.

## 9. Performance considerations

- Normal engine at Difficulty 5 is the most expensive path; must be profiled to confirm it lands within the 3–5s budget on mid-range hardware, not just high-end dev machines.
- Transposition table size must be capped (e.g. fixed number of entries with replacement scheme) to avoid unbounded memory growth across a long game or many AI-vs-AI games in one session.
- Stockfish's NNUE weight file is several MB; first Grandmaster-mode use will have a noticeable load delay — the UI must show a loading state for this specifically, distinct from the per-move thinking indicator.

## 10. Error handling

| Failure | Detection | Handling |
|---|---|---|
| Worker throws / crashes | `onerror` handler on the Worker object | Emit `{type:'error'}` equivalent to state manager → PRD §5.5 toast, freeze board, offer New Game |
| Stockfish asset fails to load | Load promise rejects / timeout | Disable Grandmaster option for session, inline message (PRD §5.5) |
| Engine returns an illegal/unparseable move | Rules-engine validation fails | Reject result, surface as worker error (do not retry silently in a loop) |
| LLM endpoint non-2xx / network or CORS error / exhausted retry | `fetch` rejects or reply unparseable/illegal after one retry | Surface via the same §5.5 worker-crash path (toast, freeze board, New Game) — `requestMove` rejection |
| Search exceeds time budget significantly | Worker-side watchdog timer | Force-return best move found so far rather than hang the request |

## 11. Testing strategy (traces to spec §10 acceptance criteria)

- **Rules engine**: unit tests against known FEN/perft positions (standard perft node-count test suite) to validate move generation correctness independent of any AI.
- **Normal engine**: regression tests on fixed positions with known best moves/tactics at each difficulty tier; confirm time/depth budgets are respected (§4.3 table).
- **Grandmaster integration**: smoke test that UCI handshake completes, a `position`+`go` round-trip returns a legal move, and load-failure handling triggers correctly when the asset URL is deliberately broken in a test.
- **End-to-end**: scripted playthroughs covering all Human/AI controller combinations including LLM-Assisted (spec AC-4, AC-8), a full AI-vs-AI game to completion (AC-3), and an offline-mode check confirming Human vs Human / Human vs Normal works with network disabled (AC-6).

## 12. Open technical risks

- Timing consistency of `go movetime` across browsers/devices for Stockfish — may need a depth-based fallback if movetime proves unreliable (noted in §5.3).
- Whether centipawn-based move-quality thresholds (PRD §5.4) feel right in practice — likely needs a tuning pass after initial playtesting, not purely a code-review concern.
- Memory/GC behavior of the Normal engine's transposition table across long AI-vs-AI spectator sessions (PRD §2.3) — worth a soak test.
- **License drift:** if a contributor later vendors the `nmrugg/stockfish.js` GPL asset into the repo (instead of CDN `importScripts`), or links `chessops`/`shakmaty` into the rules engine, the project becomes GPL-3.0 effectively. The §5.1 worker-over-UCI isolation is the guardrail; review it before any dependency add (spec §12.4).

## 13. Dependencies & references

Concrete picks binding spec §12 to implementation:

| Concern | Pick | License | Loaded how |
|---|---|---|---|
| Grandmaster engine | `nmrugg/stockfish.js`, single-threaded flavor (`npm i stockfish`) | GPL-3.0 | CDN `importScripts` in the GM worker, lazy (§5.1) |
| Rules engine | In-house 0x88 (FR-1, §3) | n/a | Inline in the single HTML file |
| Rules fallback | `chess.js` | permissive (verify in-repo) | CDN, only if §11 perft tests fail |
| Normal engine | In-house (FR-3, §4) | n/a | Inline |

Study references (read, don't import): **Sunfish** (§4 search/eval + difficulty-ladder roadmap), **Pleco** (bitboard data structures, MIT lib crate), **shakmaty** (bitboard/Zobrist/FEN-SAN-UCI correctness patterns), **chessops** (typed rules). License policy and the full option table live in spec §12; this TDD section only names the binding choices.
