# Piero's AI Chess, 2026 Edition — Specification

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
- **G2.** Let the user independently assign White and Black to Human, Browser-AI (with a difficulty), Grandmaster AI, or LLM-AI (with its own endpoint/model config).
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
| Browser-AI | Local custom engine; requires a **Difficulty** sub-selection |
| Grandmaster AI | Stockfish WASM; selectable via Browser-AI difficulty 6–11 (full strength or UCI_Elo-limited) |
| LLM-AI | Calls a user-configured OpenAI-compatible chat endpoint per move; requires **Endpoint / API key (optional) / Model** sub-selection |

All Human/AI combinations must be supported without special-casing in the rules engine (LLM-AI counts as an AI tier):

- Human vs Human
- Human vs AI (either tier, either color)
- AI vs AI (same or different tiers) — the app must be able to play itself out, useful for demos/testing

## 5. Functional requirements

### FR-1. Chess rules engine
- FR-1.1 Legal move generation for all pieces, including castling (kingside/queenside, with all legality conditions: king/rook unmoved, no intervening pieces, king not in/through/into check), en passant, and pawn promotion (must prompt for piece choice when a human promotes; AI selects programmatically).
- FR-1.2 Check, checkmate, and stalemate detection.
- FR-1.3 Draw detection: fifty-move rule, insufficient material, and **threefold repetition** are all implemented. Threefold is detected by counting occurrences of each position (key = FEN board + side to move + castling rights + en passant square, ignoring the move counters) across a game; the 3rd occurrence of the same position → draw ("threefold repetition"). This is the rule that catches perpetual check / shuffle loops early (vs the fifty-move rule's ~100-ply wait). Tournament/match games additionally keep the 200-half-move cap (FR-9.6/FR-9.7) as a backstop for loops that never quite repeat.
- FR-1.4 **Configurable move limit (game-length cap):** the setup screen offers a **Move limit** selector (Off / 40 / 60 / 80 / 100 / 150 / 200 full moves, default Off, persisted). When set, a single game is auto-drawn at that many full moves (half-moves × 2) — a user-controlled safety net, since the fifty-move rule (FR-1.3) only fires after 100 *consecutive* quiet plies, which engine games can dodge with periodic captures/pawn moves. Off = rely on the fifty-move rule and natural endings. In Match mode the selector is shown and the cap is the lesser of the user limit and the 200-half-move safety net (so it can only tighten FR-9.7, never raise it); in Tournament mode the selector is hidden and the fixed 200-half-move cap applies.
- FR-1.5 A move is only committed to the game state if fully legal; illegal move attempts by a human are rejected with visual feedback and do not change state.
- FR-1.6 The engine must produce a FEN string for any position and standard algebraic notation (SAN) for any move, since both AI tiers and the move-history UI depend on them.

### FR-2. Game modes & player assignment
- FR-2.1 Before a game starts, the user selects White's controller and Black's controller independently, per §4.
- FR-2.2 If Browser-AI is selected for a side, a Difficulty must also be selected (see FR-3.2).
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
  | 6 — Grandmaster (Elo 1350) | Stockfish with `UCI_LimitStrength`=true, `UCI_Elo`=1350 | Best move only (Elo-limited Stockfish) | ~1350 |
  | 7 — Grandmaster (Elo 1800) | Stockfish with `UCI_LimitStrength`=true, `UCI_Elo`=1800 | Best move only (Elo-limited Stockfish) | ~1800 |
  | 8 — Grandmaster (Elo 2200) | Stockfish with `UCI_LimitStrength`=true, `UCI_Elo`=2200 | Best move only (Elo-limited Stockfish) | ~2200 |
  | 9 — Grandmaster (Elo 2600) | Stockfish with `UCI_LimitStrength`=true, `UCI_Elo`=2600 | Best move only (Elo-limited Stockfish) | ~2600 |
  | 10 — Grandmaster (Elo 3000) | Stockfish with `UCI_LimitStrength`=true, `UCI_Elo`=3000 | Best move only (Elo-limited Stockfish) | ~3000 |
  | 11 — Grandmaster full | Stockfish NNUE | Best move only (Stockfish) | Full strength |

  Levels 6–11 route to Grandmaster (`type: 'grandmaster'` in ControllerConfig) with an optional `elo` field. The Browser-AI difficulty picker exposes them (PRD §2.1) to keep the controller picker to three choices. Levels 1–5 remain the Normal engine's own depths.

  Exact ELO targets are a PRD/TDD tuning concern; this table constrains *mechanism* (how difficulty is achieved), not exact numbers.
- FR-3.3 The Normal engine must never crash or stall the worker on any legal position, including positions with very few legal moves (forced sequences) or none (must correctly report game-over rather than search).
- FR-3.4 Evaluation function must include, at minimum: material balance, piece-square tables (tapered between middlegame/endgame), mobility, and basic king safety.

### FR-4. AI — Grandmaster mode
- FR-4.1 Uses Stockfish compiled to WebAssembly, communicated with via the standard UCI text protocol.
- FR-4.2 Runs at effectively unrestricted strength (no artificial ELO cap) within a bounded thinking time per move, configurable as a constant (not user-facing in v1).
- FR-4.3 If the Stockfish WASM asset fails to load (e.g. opened via `file://` which blocks the Worker — see NFR-5.2; missing sibling files; blocked CDN), the app must degrade gracefully: surface a clear, actionable error to the user (including the HTTP-server workaround when the cause is `file://`) and prevent selecting Grandmaster mode, rather than silently failing mid-game.
- FR-4.4 Grandmaster engine selection must not require any server-side component — asset loading is client-side only (script tag / fetch from CDN).

### FR-5. Board & interaction UI
- FR-5.1 Rendered 8×8 board, correctly oriented (can be flipped), with legal-move highlighting when a human selects a piece. **Coordinate labels (a–h, 1–8) on the board edges are toggleable** via a persisted 'Show coordinates' option (on by default).
- FR-5.2 Move input via click-to-select-then-click-to-move **and**
  drag-and-drop. Both paths share one move-resolution routine
  (`attemptMoveTo`), so promotion and legality behave identically; an invalid
  drop simply snaps back (no move played, selection cleared).
- FR-5.3 Last-move highlight, check indicator, and captured-piece tray for both sides.
- FR-5.4 Move history panel in SAN, scrollable, updated after every ply.
- FR-5.5 Plain-English move narration, shown and optionally spoken. After every move a small toaster under the board shows a **locally generated** plain-English description (e.g. "White: Knight to f3", "Black: Pawn captures on d5", "Castles kingside", "Queen captures on f7 — checkmate!"), built from the pre-move board + the move + SAN — no LLM call; inaccuracies/mistakes/blunders append a short cue. When the game ends the same toaster appends the end-game verdict (the summary's "who played cleaner" sentence) on a second line. A **Speak moves** toggle (off by default, persisted) speaks each narration aloud: it POSTs `{model,input,voice,response_format}` to a user-configured **OpenAI-compatible `/v1/audio/speech`** endpoint (e.g. a local Kokoro server such as docker-kokoro — config: base URL, model, voice, persisted) and plays the returned audio blob, **falling back to the browser's native `speechSynthesis`** when no endpoint is set or a request fails. Speech is cancelled on toggle-off, new game, and entering replay; a newer move supersedes any in-flight fetch/utterance. The only TTS protocol referenced is "OpenAI-compatible /v1/audio/speech" (vendor-neutral, §1).

- FR-5.6 **Audio feedback** (all off by default, persisted, vendor-neutral — §1):
  - **Piece sounds** ("Sound effects" toggle): move / capture / check / game-end cues, played as **sampled CC0** clips (lichess `standard`/`sfx` themes) embedded as base64 data URIs in a `#sound-clips-src` block — self-contained, no network. (An earlier synthesized Web-Audio version was replaced after playtest feedback.)
  - **Spectator reactions** ("Spectator reactions" toggle, AI-vs-AI only): crowd clips keyed off the move-quality tag — `best`→ applause, `blunder`→ boo, `mistake`→ disappointment, `inaccuracy`→ shocked, checkmate→ cheer — **sampled real crowd audio under the Pixabay License** (no attribution, GPL-3.0-compatible), embedded as base64 in `#reaction-clips-src`. Reactions **fire only on captures, gated by a crowd-enthusiasm threshold**: each move gets a drama score (captured-piece value + eval swing in pawns + a quality premium) and fires only if it beats a linear threshold that **decays with ply** — the crowd starts quiet (early game, only queen/rook captures or big swings fire) and grows more invested (by the endgame any capture gets a reaction), avoiding constant applause in strong matches. Clips fade out over their last 400 ms.

### FR-6. Game controls
- FR-6.1 New Game (re-opens controller/difficulty selection).
- FR-6.2 Resign (available to a human player, ends game with the opponent as winner).
- FR-6.3 Flip board.
- FR-6.4 Pause/stop AI-vs-AI games (since these run unattended). A spectator **speed control** (Very slow / Slow / Normal / Fast / Instant) sets the pause between AI-vs-AI moves purely for watchability — the AI still computes at its real budget; persisted (PRD §2.3).
- FR-6.5 Destructive actions are confirmed. New Game, Rematch, and Resign
  during an in-progress game prompt a native `confirm()`; if the action would
  discard an in-progress game with moves, OK saves it as a PGN file first
  (FR-8.2) then proceeds, Cancel keeps playing. Resign only confirms (it
  completes the game, which can then be saved from the summary). The spectator
  **Stop** control (ends the current game / series early with no winner) also
  confirms before acting. Additionally, an accidental page close / refresh
  during an active game or series triggers the browser's native "leave site?"
  guard (a `beforeunload` handler, gated so it only fires when there is a
  game or series in flight to lose).
- FR-6.6 Optional chess clock (FIDE-style): a selectable time control at
  setup — Off (default) / Bullet 1+0 / Blitz 3+2 / Rapid 10+5 / Classical
  30+0 — gives each side a total time budget plus a per-move increment. The
  side-to-move's clock runs while its move is being decided (human
  deliberation or engine think); the AI-vs-AI spectator pacing delay does
  NOT count (that is viewing time, not thinking time). On each completed
  move the mover's clock gains the increment. A clock reaching zero ends
  the game as a loss on time for that side (FR-7.2). Applies to all
  controller types; engines rarely flag (their think time is small relative
  to a minutes-long budget), so in practice it constrains humans and slow
  LLMs. Selection persists across sessions.
- FR-6.7 Cross-hardware fairness: Browser-AI uses node budgets (not wall-clock
  time — see TDD §4.3) so the same difficulty level searches the same tree
  on any CPU. Grandmaster (Stockfish) exposes selectable Elo levels via
  `UCI_LimitStrength` + `UCI_Elo` (difficulty 7–9, levels below full
  strength at 6), making Elo-limited Stockfish play the same on fast and
  slow machines alike.
- FR-6.8 **Take over a side mid-game**: a "Take over…"
  button — shown in an active game (hidden only during replay or once the game
  is over) — opens a modal that reuses
  the setup screen's side-config UI under a `.takeover-config` class. The
  spectator picks which side to replace and configures its new controller
  (Human / Browser-AI at any level / LLM-AI with any endpoint+model+persona,
  validated like the setup screen — LLM needs base URL + model, no Grandmaster
  from file://, no mixed-content). On confirm, `matchConfig[side]` is swapped
  and play resumes from the exact current position (the position, move
  history, captured pieces, eval bar and clocks are unchanged; `playTurn`
  reads the controller live each turn). The new config is mirrored back to the
  setup side-config so Rematch / next-game defaults keep it. The end-game
  summary mixes moves from the old and new controller for the taken-over side.
  **Series end-and-continue:** during a tournament/match, confirming a take-over
  ENDS the series — the partial results (the games completed so far, i.e. the
  old matchup's record) are sealed and saved to the results file, the
  in-progress game is not counted, and play continues as a single game from
  the exact current position with the swapped side. This keeps the series
  stats/Elo honest (they describe only the fixed original matchup) while still
  letting a spectator step into an interesting position to play it out.

### FR-7. Game status & notation
- FR-7.1 Persistent status line reflecting game state: whose turn, check, checkmate, stalemate, draw (with reason).
- FR-7.2 On game end, display the result (1-0, 0-1, ½-½) and reason. Reasons include checkmate, stalemate, draw (fifty-move rule / threefold repetition / insufficient material), move-limit draw (FR-1.4), resignation, and — when the chess clock is enabled (FR-6.6) — loss on time.

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

### FR-9. AI — LLM-AI mode
- FR-9.1 On selecting LLM-AI for a side, the user supplies, per side: an **API base URL** (OpenAI-compatible, stored as `apiBase`), an optional **API key** (blank works for local servers like Ollama / LM Studio), a **Model** name, and a **System prompt** (stored as `systemPrompt`). Both `apiBase` and `model` are required to start. The two sides may use different endpoints/models/prompts. The system prompt is chosen from a set of built-in personas (Grandmaster / Tactician / Attacker / Positional / Think-then-commit) via a dropdown that loads the preset into an editable textarea, or typed freely as a custom prompt; whichever text ends up in the textarea is what is sent (persisted with the rest of the config). Every prompt keeps the iron output contract — exactly one legal UCI move — so the parser stays reliable; the move extractor resolves to the **last** legal-move token mentioned (so a chain-of-thought "Think-then-commit" reply that names candidates then commits resolves to the choice, not a rejected candidate). A **Saved profile** dropdown above these fields lists every URL+key+model+persona combination that has passed the Verify gate (the per-side **Verify** button or the automatic pre-start probe — both run the same one-shot probe); selecting one fills all four fields (still editable afterwards), or **Custom…** leaves them blank for fresh entry. Profiles live under their own `localStorage` key and are shared by both sides.
- FR-9.2 Each move is requested via `POST {apiBase}/chat/completions` (Authorization: Bearer {apiKey} when a key is supplied), sending the current FEN, the side to move, recent SAN move history, and the full list of legal UCI moves, and asking the model to return exactly one move from that list (constrained choice — the main reliability lever for legal LLM chess).
- FR-9.3 The reply is parsed for one UCI move (preferring an exact legal-move token) and **re-validated through the rules engine** like every other AI move (NFR-7.2). An illegal/unparseable reply is retried **up to once** with a corrective message that feeds the bad reply back into the conversation. If it still fails, or the network call itself fails, the side **falls back to a quick local Normal-engine move (difficulty 3) so play continues**, with a toast — LLM APIs fail transiently far more often than a local engine, and per the primary goal (entertain) a flaky API must not kill the game. Only if the fallback itself errors does the game end.
- FR-9.4 Setup **config** (per-side controller, difficulty, and LLM `apiBase`/`apiKey`/`model`) is **persisted to `localStorage` and proposed as defaults on the next restart** on the same computer. This is the single permitted persistence exception: it covers setup selections only, **not** game state/history/ratings (those remain barred by NG2 / FR-8.1). Persistence is best-effort and silently no-ops when storage is unavailable (e.g. disabled, or some `file://` contexts). Note the key is stored locally and readable by anyone with browser-profile access — acceptable for a single-user local tool, not for sharing the file with a real key typed in. Separately, every LLM config that passes the Verify gate is also remembered as a reusable **profile** (URL+key+model+persona, keyed by `apiBase`+`model`) under its own `localStorage` key and listed in a per-side **Saved profile** dropdown (FR-9.1) — re-verifying a known endpoint+model updates the profile in place rather than duplicating it.
- FR-9.5 The eval bar and move-quality tags (PRD §5.4) remain functional in LLM-AI mode: the Normal engine exposes an `analyze` message (spec §9) that grades *any* played move — LLM or human — against a quick local search, returning `evalCp` (the played move's eval) and `bestEvalCp` (the best line's eval). The chat endpoint returns no eval, so the local engine acts purely as a judge, never as a mover — giving one yardstick to compare LLM moves against Normal and Grandmaster.
- FR-9.6 **Tournament mode (LLM gauntlet):** when exactly one side is LLM-AI, the setup screen's **Mode selector** (Single game / Tournament / Match — see FR-9.7) offers **Tournament**; choosing it renames the Start button to "Run tournament" and launches the gauntlet instead of a single game. (Replaces the earlier "Tournament mode" checkbox; the selector also hosts Match mode, FR-9.7. Tournament always greys out the Time control — it runs without a chess clock; Match honors the Time control as a free choice (default Off) — see FR-9.7.) It plays a 3-game round between the LLM and the AI opponent at each difficulty (1–11) using an **adaptive gauntlet** (cf. the LLM-Chess benchmark's level-selection heuristic): after each 3-game round, if the **AI swept** (LLM lost all 3) the tournament **stops** — that level is the LLM's floor; otherwise (LLM swept **or** the round was mixed) it **climbs to the next difficulty** until the top level (Grandmaster full) is reached. (The earlier "any 3-0 sweep ends it" rule ended a strong LLM at level 1 and never found its ceiling; the adaptive rule is what makes the Elo estimate below resolvable.) **Difficulty 1 is a pure random player** (it picks uniformly from all legal moves — the weakest opponent, the same baseline as the benchmark's random-agent gating phase). Colors alternate each game for fairness; tournament games run **without a chess clock** so the LLM is not penalized for slow inference / 429-5xx retry backoff, and are **capped at 200 half-moves → draw** (the benchmark's 200-move cap) as a backstop safety net; threefold repetition (FR-1.3) is also detected, so perpetual-check loops draw at the 3rd repetition well before the cap. LLM 4xx/5xx errors use the exponential-backoff retry loop (the local fallback move of FR-9.3 is only the last resort). On completion, a per-level results table (W-L-D from the LLM's perspective + outcome) **plus the diagnostic blocks below** are shown and auto-saved to a timestamped text file (`tournament-{model}-{yyyymmdd-hhmmss}.txt`), ending with a **methodology footnote** noting that our single-turn constrained-choice protocol is simpler than the benchmark's multi-turn agentic Game Duration (so our instruction-following % is not directly comparable) and that our Elo pool (Stockfish `UCI_Elo`) differs from theirs (Dragon/chess.com) though the MLE method is identical:
  - **Estimated Elo (LLM-Chess-leaderboard-style):** a maximum-likelihood Elo rating is computed from the (opponent Elo, game score) pairs of the Stockfish `UCI_Elo`-anchored games (levels 6–10: 1350/1800/2200/2600/3000 — level 11 full and Normal 1–5 have no anchor and are excluded). It solves `Σ(Sᵢ − Eᵢ(R̂)) = 0` with `Eᵢ(R)=1/(1+10^((Rᵢ−R)/400))` (bisection), and reports a 95% CI from the Fisher information `I=ΣE(1−E)(ln10/400)²`, `SE=1/√I`. Extreme cases report bounds: all-wins → "above {max anchor}", all-losses → "below {min anchor}"; if the LLM never reached the Stockfish levels, it reports "insufficient Elo-anchored data". Annotated as the Stockfish `UCI_Elo` pool (not cross-pool-comparable to chess.com/FIDE, per the benchmark's own caveat).
  - **Instruction-following stability** (the benchmark's headline metric): counts of LLM moves that were legal on the first try, needed the corrective retry, or fell back to a local Normal move — reported as "% own legal moves" (our honest analog of their "Game Duration" / instruction-error rate, since our fallback keeps the game alive rather than losing it).
  - **LLM move-quality distribution**: the existing best/good/inaccuracy/mistake/blunder tags (FR-9.5), accumulated across the LLM's own moves into percentages.
  - **Game-level + per-move efficiency stats**: games played, average moves/game, average LLM inference latency per move (excluding the local analysis), and average completion tokens per move (from the endpoint's `usage`).
  A spectator Stop ends the tournament early and shows the partial table.
- FR-9.7 **Match mode (best-of-N head-to-head):** the Mode selector offers **Match** when **both sides are non-human** (typically two LLM-AI sides — different models/endpoints, or the same model with different personas to A/B-test system prompts; a Browser-AI/Stockfish side is also allowed). The user picks an **even game count** (6 / 10 / 20 / 40, default 10); Start becomes "Run match". It plays that many games between the two configured sides as-is (no level sweep — that is the tournament's job, FR-9.6), **colors alternating each game** so each side plays White exactly N/2 times. A chess clock is **optional** (a free choice, default Off; the Time control is honored even with an LLM side, at the user's discretion); games are **capped at 200 half-moves → draw** (same safety net as the tournament), use the same spectator pacing / Pause / Stop, and LLM errors use the same retry/fallback as a normal game. The output is a **relative** strength ranking (NOT an absolute Elo — there is no anchor, unlike the tournament's Stockfish-Elo MLE), shown and auto-saved to `match-{labelA}-vs-{labelB}-{yyyymmdd-hhmmss}.txt`:
  - **Result:** W-draw-L from side A's perspective (A = configured White, B = configured Black; identity fixed, colors alternate), plus each side's score (points/N).
  - **Relative Elo difference:** `Δ = −400·log10(1/scoreA − 1)` from A's expected score (the standard logistic expected-score formula `E=1/(1+10^(−Δ/400))` used in engine testing — a *gap*, not a rating, so not comparable to the tournament's anchored MLE Elo or to FIDE). A shutout (one side won every game) reports the gap as unbounded. `scoreA` is wins + ½·draws over N.
  - **Likelihood of Superiority (LOS):** the Bayesian posterior `P(A is truly stronger)` from the decisive games only (draws carry no superiority signal), with a uniform prior → A's win-probability posterior is `Beta(W+1, L+1)` and `LOS = Σⱼ₌ₗ₊₁..W+L+1 C(W+L+1, j)·0.5^(W+L+1)`. Reported as a %; conventionally >95% is "significant". `null` → "inconclusive (no decisive games)" when all games drew.
  - **Per-side diagnostics:** the move-quality distribution (FR-9.5) for **both** sides (not just LLMs — the match scores every non-human side so the head-to-head is fair regardless of controller type); LLM sides additionally get instruction-following % (first-try / retried / fallback), average inference latency, and average completion tokens. Side labels are `model [persona]` for LLMs (so same-model/different-prompt matches are legible), else the engine name.
  - A **methodology footnote** notes the Elo model (relative gap always; absolute MLE additionally when a Stockfish side anchors), the LOS definition, the single-turn constrained-choice protocol, and that small N is noisy (N≥20 recommended for a stable read).
  A spectator Stop ends the match early and shows the partial table. **Absolute Elo when one side anchors:** if exactly one side is a Stockfish level with a known UCI_Elo (difficulty 6–10; level 11/full has no number and cannot anchor), that side is a *fixed* anchor — every game is the other side vs that one Stockfish Elo — so the match runs the **same MLE** as the tournament (FR-9.6) over the synthesized `{elo, score}` games (score from the non-anchor side's W/draw/L) and reports an **absolute Elo ± 95% CI** for the non-anchor side, in the **same Stockfish UCI_Elo pool as the tournament** (directly comparable — unlike the relative gap). All-win/all-loss report "above/below" the anchor rather than a finite estimate. When neither or both sides anchor (the common LLM-vs-LLM or LLM-vs-Browser-AI case), only the relative gap + LOS are shown.

### FR-9.8. Configurable reasoning level (per LLM-AI side)
- A **Reasoning** dropdown (Off / Low / Medium / High, default Off, persisted) sets the `reasoning_effort` request parameter for that side's LLM moves — letting a user trade latency/cost for move quality, directly serving the secondary goal (how much does extra reasoning help an LLM *play chess*?). The parameter is **endpoint/model-specific** (only reasoning-style models honor it), so it is sent opportunistically with **graceful degradation and no capability probe**: if the endpoint returns HTTP 400 while `reasoning_effort` was sent, the side marks it unsupported for the rest of the game and retries that move without it — so an unsupported knob never breaks the constrained-choice contract (FR-9.2/9.3) or poisons eval data. The chosen level is shown in match/tournament side labels (`model [persona, reasoning:high]`) so an A/B comparison is legible. It rides the existing LLM retry/backoff path (FR-9.3/9.6); the `Verify` probe does not send it (connectivity check only), so the first real move discovers support. *(Benchmark caveat, kept honest: more reasoning has historically helped only modestly and sometimes degraded play — this knob exists precisely so that claim can be tested per-model.)*

### FR-9.9. Hint coach (human-requested move suggestion)
- A human player may click "💡 Hint" during their turn to ask an LLM coach for a move suggestion — up to **3 hints per game** (a fixed budget, decremented and shown in the button label, reset on new game / take-over; not difficulty-gated, since a hint is a learning aid and the opponent's strength is a separate axis). The coach is its **own** OpenAI-compatible chat endpoint (base URL + key + model), configured under a "💡 Hint coach" panel on the setup screen and persisted (FR-9.4's prefs) — **independent of the opponent's controller**, because a Human-vs-Browser-AI/Grandmaster game has no LLM endpoint of its own. The coach is prompted to return the best move (UCI) on line 1 and a one-line reason on line 2; the move is parsed from the **first line only** (so a reason that names another move cannot pollute the pick) and re-validated through the rules engine (NFR-7.2). The suggestion is rendered as a **green ring on its from/to squares** and the reason is shown in the narration toaster (FR-5.5) — it is **never auto-played** (the human still moves themselves). A failed/empty/illegal coach reply alerts the user and **does not consume a hint**. The button appears only on a human's turn when a coach is configured and hints remain. Rationale: serves the primary goal (entertain / make a game winnable for a beginner) directly.

### FR-9.10. AI commentary per move
- A "🗣 AI commentary" toggle (off by default, persisted) has the **AI assistant** (FR-9.9's shared endpoint) generate ONE short, entertaining sentence after each move — shown as a second line in the narration toaster (FR-5.5) and **spoken via TTS when "Speak moves" is on** (in which case the factual narration is shown but *not* spoken, so only the comment is voiced — one voice, not two). It is **best-effort and non-blocking**: a single fetch with **no retry loop** (a transient failure silently drops that move's comment — a retrying call per move would pile up in AI-vs-AI), and a **generation token drops any comment superseded by a newer move** (fast games skip stale commentary instead of queueing it). It never blocks the game loop or AI-vs-AI pacing. Rationale: pure entertainment (the "hear the game" angle); the factual move narration (FR-5.5) already covers the informational need, so this is optional color on top.

### FR-9.11. In-session result tally
- A **running scoreboard** (games played, White wins / draws / Black wins) accumulates across every completed **single** game in the page session — shown in a small panel below the board, with a reset link. It counts decisive games and draws only; an aborted or spectator-Stopped game (result `*`) is **not** counted. It **excludes tournament/match games** (those series own their own results panel — FR-9.6/FR-9.7). In-memory only (no persistence); reset on page reload or via the link. Rationale: a spectator (AI-vs-AI) or a human (Human-vs-AI) can see their record across the session without the overhead of configuring a full series.

### FR-9.12. AI move time cap
- A setup-screen "AI move cap" selector (Off / 15s / 30s / 60s / 120s, default Off, persisted) sets a hard ceiling on any single **AI** move's think/network time. On expiry the timed-out move throws, routing through the existing per-controller failure path: an **LLM falls back to a quick local move** (FR-9.4 — so a hung endpoint can't freeze the game or stall a tournament/match gauntlet); an **engine** timeout is pathological (engines are already bounded — node budgets with a 30s cap, Stockfish `movetime 3000`) and ends the game. It applies to **AI moves only** — a human's deliberation is governed by the chess clock (FR-6.6), not a per-move ceiling (a per-move cap on a human is non-standard and harsh). Rationale: a robustness guard against hung/slow endpoints, complementing — not replacing — the total chess clock.

### FR-9.13. Interface language (EN/IT)
- A 🌐 Language selector on the setup (landing) page chooses the interface language — **English (default) or Italian** — persisted to prefs. It applies app-wide via a single global setting (not per-side — the two-players-one-screen case is out of scope). On change the page reloads, which overlays Italian onto the English DOM source. **Tier B scope:** localized are the static chrome (setup screen, game screen, takeover modal), the move narration (`describeMove`), the status line, the thinking indicator, the Pause/Resume button, and game-over reasons. **Left in English** (documented boundary): tooltips (`title=`), long hint paragraphs, toast messages, and the generated tournament/match result tables — hover-only, transient, or complex generated prose. Rationale: an Italian user gets a localized app without the cost/risk of a full per-string pass; the most-seen text is localized.

## 6. Non-functional requirements

### NFR-1. Single-file architecture
The deliverable is one `.html` file containing all CSS and JS inline. Exception: the Stockfish WASM engine and its network weight file may be loaded from an external CDN at runtime (see NFR-5) — this is the one permitted external dependency, since embedding a multi-megabyte WASM binary as base64 is impractical and against the spirit of an editable single-file app.

### NFR-2. Performance
- NFR-2.1 UI interactions (piece selection, highlighting, menu navigation) must respond within 100ms regardless of AI activity.
- NFR-2.2 Browser-AI must respect its difficulty's time/depth budget (§FR-3.2) and not exceed it by more than a small margin.

### NFR-3. Responsiveness (non-blocking UI)
- NFR-3.1 Both AI tiers must run their search inside a Web Worker. The main thread must never run a blocking search loop.
- NFR-3.2 The board and controls must remain interactive (e.g. flipping the board, scrolling move history) while an AI is thinking.

### NFR-4. Compatibility
- NFR-4.1 Must run in current versions of Chrome, Firefox, Safari, and Edge without polyfills for standard Web Worker / WebAssembly APIs.
- NFR-4.2 Must be usable on both desktop and mobile viewport widths (responsive layout).

### NFR-5. Network dependency (Grandmaster and LLM-AI modes)
- NFR-5.1 Human vs Human and Human vs Browser-AI must work fully offline once the page is loaded.
- NFR-5.2 Grandmaster runs **fully offline once served**: the two-file Stockfish asset bundle (`stockfish-18-lite-single.js` + `stockfish-18-lite-single.wasm`) is **committed to the repo** next to `chess.html` (GPL-3.0 — §12.4), so no internet download is ever needed. The loader tries the local bundle first, then falls back to a CDN; the CDN is currently non-functional anyway (jsDelivr refuses the >150 MB npm package with HTTP 403), but the committed bundle makes that irrelevant. **Serving caveat:** Grandmaster must be served over **HTTP(S)**, not opened via `file://`. Browsers block `new Worker()` from a `file://` page (same-origin policy on local files), and the Stockfish loader runs as a Worker. (Human-vs-Human and Human-vs-Normal do work via `file://` — see NFR-5.1 — because the Normal engine runs from an inline Blob URL.)
- NFR-5.4 LLM-AI mode requires network access to a user-supplied OpenAI-compatible endpoint on every move. The endpoint must itself serve permissive CORS headers (the browser makes the call directly); local servers such as LM Studio, llama.cpp, vLLM, and LocalAI do, while OpenAI's hosted API does not from a browser — a CORS-friendly proxy is the user's responsibility. No CORS handling is implemented app-side.
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
- AC-6: The app loads and is fully playable (Human vs Human, Human vs Browser-AI) with no network connection.
- AC-7: A Stockfish load failure is handled gracefully and does not break the rest of the app.
- AC-8: LLM-AI mode produces legal moves from a configured OpenAI-compatible endpoint; an endpoint error or illegal reply (after retry) triggers a local fallback move rather than ending the game, and only a fallback failure freezes the UI gracefully.

## 11. Assumptions & open questions (to be resolved in PRD/TDD)

- A11.1 Exact ELO targets per Normal difficulty level — needs tuning/benchmarking, not specified here.
- A11.2 Whether Grandmaster mode exposes a "thinking time" or strength setting to the user, or is fixed — currently spec'd as fixed max strength.
- A11.3 ~~Whether multi-threaded WASM (requiring specific server headers) is a hard requirement or a "best effort if headers present"~~ — **Resolved (see §13):** use a single-threaded Stockfish WASM build for v1. Recommended package: `nmrugg/stockfish.js` single-threaded flavor (`stockfish` on npm). Multi-threading is a v1.1 upgrade gated on confirming the deployment host can serve COOP/COEP headers. See §13 and TDD §5.2.
- A11.4 Drag-and-drop move input — **shipped (2026-07-05)** as part of FR-5.2; see that FR.
- A11.5 Visual theme and animations are UX decisions left to the PRD. (Sound effects and spectator reactions were originally deferred here too but are now fully spec'd — FR-5.6.)
- A11.6 LLM-AI prompt robustness — the single-hard-move prompt + one-retry policy (FR-9.3) is a baseline; real-world move legality/parse rates depend on the chosen model and may need a richer prompt or constrained decoding later. Not blocking v1.

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

**Decision:** `nmrugg/stockfish.js` single-threaded. It carries full NNUE evaluation (so strength is unbounded relative to the Normal engine and well beyond human-Kasparov level) while running without COOP/COEP — directly satisfying NFR-5.3's "single-file app must work on any static host" constraint. NPS is capped to one core, an accepted trade-off (TDD §5.2). Note this does NOT extend to `file://`: Grandmaster still needs HTTP serving (NFR-5.2), because the Stockfish Worker cannot be created from a `file://` page.

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
- **The Stockfish runtime is vendored into the repo** as two sibling files (`stockfish-18-lite-single.{js,wasm}`) next to `chess.html`, so Grandmaster runs offline from the local bundle once served over HTTP — on the live site or a local server (NFR-5.2; `file://` is blocked because the Stockfish Worker can't load from a local-file page). This is a deliberate licensing choice: because Stockfish (GPL-3.0) is a derivative/combined-work component, the project embraces GPL-3.0 copyleft rather than trying to keep it at arm's length.
- Most other strong chess libs (`shakmaty`, `chessops`) are also GPL-3.0 but remain **design references only** (the rules engine is hand-rolled 0x88 — FR-1, no GPL `chessops`/`shakmaty` linkage). They could be vendored later without further license impact; today they are not.
- The prior (Apache-2.0) posture kept Stockfish as a UCI-over-worker runtime dependency and forbade vendoring it; that constraint is **lifted** by this relicense.

### 12.5 Summary decisions
| Concern | Decision |
|---|---|
| Grandmaster engine | `nmrugg/stockfish.js`, single-threaded flavor, loaded from CDN via worker `importScripts` |
| Rules engine | Hand-rolled 0x88 (FR-1); `chess.js` fallback; `shakmaty` as design reference |
| Normal engine | In-house (FR-3); Sunfish/Pleco as study references |
| License posture | Whole project GPL-3.0; Stockfish runtime vendored into the repo (`stockfish-18-lite-single.{js,wasm}`) so Grandmaster runs offline once served over HTTP |

## 13. Glossary

- **SDD** — Spec-Driven Development.
- **PRD** — Product Requirements Document (derived from this spec; defines user-facing flows and UX).
- **TDD** — Technical Design Document (derived from this spec; defines architecture, modules, and data structures).
- **FEN** — Forsyth–Edwards Notation, a compact text representation of a chess position.
- **SAN** — Standard Algebraic Notation, e.g. `Nf3`, `O-O`, `exd5`.
- **UCI** — Universal Chess Interface, the text protocol Stockfish and most engines speak.
- **NNUE** — Efficiently updatable neural network, Stockfish's modern evaluation approach.
