# AGENTS.md — Piero's AI Chess, 2026 Edition

Operating manual for AI agents (and humans) working in this repo. Read this
first; it encodes the vision, the architecture, the tech stack, and the
standards this project is held to. Everything here is grounded in the actual
source — when code and docs disagree, **code is the source of truth** and the
docs are the thing that's wrong (see §7).

---

## 1. Vision

A chess application delivered as a **single self-contained HTML file** (inline
CSS + JS, no build step, no backend, no install) that runs in any modern
browser. Two goals, in priority order (PRD §1):

1. **Entertain.** A beginner should have a winnable, enjoyable game; a strong
   player should get a genuine challenge.
2. **Evaluate AI "intelligence."** The app is a lens for comparing engines and
   LLMs — how Normal behaves at each difficulty, how strong Grandmaster is, and
   concretely how each thinks move-to-move. This goal drives the transparency
   features (eval bar, move-quality tags, thinking indicator, AI-vs-AI
   spectating, **tournament gauntlet with an Elo estimate**) that a plain
   "let me play chess" app wouldn't need. The tournament's metrics and Elo
   method are aligned with the public **[LLM-Chess](https://github.com/maxim-saplin/llm_chess)**
   benchmark (same MLE Elo; we annotate honestly where our pool/protocol
   differs — see §4.6).

The design is deliberately **vendor-neutral**: the only cloud-LLM brand
referenced anywhere is OpenAI, and only as the de-facto *protocol* name
("OpenAI-compatible") plus placeholder text. No Anthropic/Claude/Gemini/etc.
appear, and none should be added — LLM play goes through a user-supplied
endpoint, ideally a local server.

---

## 2. Repository layout

```
chess.html                    # The app. Single file. This is the deliverable.
stockfish-18-lite-single.js   # Stockfish loader (vendored, GPL-3.0)
stockfish-18-lite-single.wasm # Stockfish NNUE WASM (vendored, GPL-3.0)
README.md                     # Project README
LICENSE                       # GNU GPL v3.0 (whole project, incl. vendored Stockfish)
AGENTS.md                     # This file (operating manual) — at root for agent tooling
specs/                        # Governing documentation (SDD layers)
  chess-app-spec.md           # SDD Layer 1 — spec (requirements, the contract)
  chess-app-prd.md            # SDD Layer 2 — PRD (UX, flows, behavior)
  chess-app-tdd.md            # SDD Layer 3 — TDD (architecture, design, schemas)
docs/                         # User-facing pages (blog / guide / kids, EN + IT)
  onefile-chess-*.html
```

No `package.json`, no build tooling, no test runner — by design. The project
is **GPL-3.0** and **vendors** the Stockfish runtime (the two sibling files
above) so Grandmaster runs offline from the local bundle — but it must be
**served over HTTP**, not opened via `file://` (see §8).

---

## 3. Tech stack

- **Single HTML file.** All CSS in one `<style>`, all JS in one `<script>`.
  No bundler, no framework, no transpile step. Edit and reload.
- **Plain DOM.** No React/Vue/etc. Board = grid of `<div>`/`<button>` squares.
- **Web Workers.** The Normal engine and Stockfish run off the main thread so
  the UI never blocks. Worker source is built by concatenating two inline
  `<script type="text/plain">` blocks (`#rules-src`, `#normal-engine-src`)
  into a `Blob` URL — keeps the engine source in the same file, versionable,
  no separate `.js` files.
- **Stockfish WASM**, single-threaded `nmrugg/stockfish.js` flavor
  (`stockfish@18.0.0`), **vendored** next to `chess.html`
  (`stockfish-18-lite-single.{js,wasm}`) and loaded as a local Worker (a CDN
  fallback exists but is currently broken/irrelevant — the local bundle is the
  path). Single-threaded on purpose: no COOP/COEP/`SharedArrayBuffer`
  requirement, so the file works on any static host. Multi-threading is
  deferred (spec A11.3, TDD §5.2; see §9).
- **Hand-rolled 0x88 rules engine** (`Rules` namespace) — legal move
  generation, FEN/SAN, check/checkmate/stalemate, draws (fifty-move,
  insufficient material, **threefold repetition**).
  `chess.js` is the documented fallback (spec §12.2) only if perft tests fail.
- **`fetch()` to an OpenAI-compatible chat endpoint** for LLM-AI mode,
  straight from the main thread. No SDK, no proxy.
- **Sampled audio**, all `HTMLAudioElement`-based (no `AudioContext`): CC0
  piece sounds (lichess) and Pixabay-License crowd reactions, embedded as
  base64 data URIs in `#sound-clips-src` / `#reaction-clips-src` blocks so the
  file stays self-contained and offline.

---

## 4. Architecture (chess.html)

The file is one big `<script>` after the markup. Top-down:

### 4.1 Rules engine (`#rules-src`)
Pure functions, no DOM/worker dependency. Owns the 0x88 board, move
generation, FEN/SAN, game-end detection. Key surface: `Rules.newGame`,
`genLegalMoves`, `applyMove`, `toFEN`/`fromFEN`, `toSAN`, `gameStatus`,
`sqName`/`nameToSq`. Reused inside the worker, so it lives in
the inline `#rules-src` block concatenated into the worker blob. Detects
checkmate, stalemate, fifty-move rule, insufficient material, and
threefold repetition.

### 4.2 Normal engine (`#normal-engine-src`)
Hand-built alpha-beta with null-move pruning, late-move reductions, a capped
transposition table, and per-difficulty move-selection noise. Difficulty table
(`DIFFICULTY`):

| Level | maxDepth | maxNodes | noise |
|---|---|---|---|
| 1 Random | 1 | 5K | wide-random (uniform over all legal moves) |
| 2 Easy | 3 | 50K | top3-random |
| 3 Medium | 99 | 200K | gaussian-small |
| 4 Hard | 99 | 800K | gaussian-tiny |
| 5 Expert | 99 | 4M | none |

Node budgets (instead of wall-clock `timeMs`) ensure the same search-tree
depth on any CPU — a fast machine reaches it faster, a slow one takes longer,
both play the same strength. A 30s safety cap prevents hangs on very slow
hardware. Runs in a worker and answers `search` (pick a move) and `analyze`
(grade an arbitrary played move — used for quality tagging of *any* move
source, and as the eval judge for LLM moves).

### 4.3 Grandmaster engine (Stockfish)
Exposed in the UI as Browser-AI **difficulty 6–11** — it builds a
`{ type: 'grandmaster' }` ControllerConfig under the hood, so the contract and
engine dispatch are unchanged; only the setup screen is simplified to 3
controller choices (Human / Browser-AI / LLM-AI). Stockfish over
UCI-in-a-worker: `uci`→`uciok`→`isready`→`readyok` handshake, then
`position fen … moves …` + `go movetime 3000` per move, parsing
`score cp`/`score mate` and `bestmove`. Loaded lazily; load failure disables
the option for the session (PRD §5.5).

**Elo levels** (`GRANDMASTER_ELO_LEVELS`): levels 6–10 set
`UCI_LimitStrength=true` + `UCI_Elo` = **1350 / 1800 / 2200 / 2600 / 3000**;
level 11 is full strength (`UCI_LimitStrength` off). These five Elo anchors are
what the tournament's Elo estimate is calibrated against (§4.6).

### 4.4 LLM-AI controller
Not a worker — a main-thread `fetch` to `{apiBase}/chat/completions`. Prompt
sends FEN + side + SAN history + the **full legal UCI move list** (constrained
choice = the main reliability lever) and asks for exactly one UCI move. A
**system prompt** is chosen per side from 5 built-in personas
(`LLM_SYSTEM_PROMPTS`: Grandmaster / Tactician / Attacker / Positional /
Think-then-commit; default Grandmaster) via a dropdown that loads the preset
into an editable textarea, or typed freely — whatever text ends up in the
textarea is sent. The move extractor resolves to the **last** legal-move token
mentioned, so a "Think-then-commit" chain-of-thought reply that names
candidates then commits resolves to the choice, not a rejected candidate.

- **Request body:** `{ model, messages }` — **no `temperature`/`top_p`/
  `frequency_penalty`**. Reasoning models (o1/o3/o4, R1, Gemini-thinking,
  Claude-thinking) reject non-reasoning params with HTTP 400, and some
  third-party OpenAI-compatible servers error on unknown params; sending them
  would make a reasoning model fail every move and silently fall back,
  poisoning eval results for a fault that isn't the model's (cf. LLM-Chess,
  Apr 2026). Rely on provider defaults.
- **Retry policy:** exponential backoff (1s → 60s cap) for **429 / 5xx**,
  retrying for up to **5 min** total; **400/401/403/404 throw immediately**
  (won't resolve); network/mixed-content errors throw immediately. There is
  **no per-request timeout** — a slow-but-successful reasoning response
  (e.g. a local model at 30 tok/s thinking 20k tokens ≈ 11 min) completes,
  matching the benchmark's rationale for a long timeout.
- **Fallback:** an illegal/unparseable reply is retried **once** with a
  corrective message feeding the bad reply back in. If it still fails, or the
  call fails, the side **falls back to a local Normal-engine move (difficulty
  3)** so play continues, with a toast. Only if the fallback itself errors
  does the game end.
- Evals for the bar/tags come from the Normal engine's `analyze` message —
  the chat endpoint returns none.

### 4.5 Game loop & UI
`playTurn()` is the spine: checks game status, dispatches to the side's
controller via the engine-agnostic `requestAIMove()` (all tiers resolve to
`{ move, evalCp, bestEvalCp }`, LLM additionally `{ llmRetries, latencyMs,
completionTokens }`), re-validates every AI move through the rules engine
before committing (NFR-7.2), then renders. AI-vs-AI games auto-play by
`playTurn()` recursing at the loop tail. Setup screen captures per-side
`ControllerConfig`; `matchConfig` is the live `{white, black}` pair.
Game-end, checked at the top of each `playTurn()`: checkmate / stalemate / insufficient material / fifty-move rule (all via `gameStatus`), plus a **move-limit draw** — a user-set cap (spec FR-1.4, default Off) in single games, tightened to the 200-half-move safety net in series (Tournament/Match).

**Cross-cutting features wired into the loop / UI:**
- **Chess clock** (FR-6.6): optional FIDE-style per-side total clock +
  per-move increment (`TIME_CONTROLS`: Off / Bullet 1+0 / Blitz 3+2 / Rapid
  10+5 / Classical 30+0, default Off, persisted). The side-to-move's clock
  runs during its move (human deliberation or engine think); spectator pacing
  delay excluded; zero = loss on time. Tournament games run **without** a
  clock (§4.6).
- **Move input:** click-to-select + **drag-and-drop** share one resolver
  (`attemptMoveTo`); invalid drops snap back (FR-5.2).
- **Confirm destructive actions** (FR-6.5): native `confirm()` on New game /
  Rematch / Resign during an in-progress game, offering save-as-PGN before
  discarding.
- **PGN save/replay** (FR-8.2): download a game as PGN; import a PGN for
  view-only replay (First/Prev/Next/Last). Standard start position only;
  loaded games don't resume engine play.
- **Spectator controls** (FR-6.4): Pause/Stop and a 5-step **speed** select
  (`SPEED_DELAYS`: veryslow 12000 / slow 9000 / normal 7000 / fast 2500 /
  instant 0 ms between AI-vs-AI moves), persisted. Render only in true
  AI-vs-AI spectating (both sides non-human).
- **Captured-piece tray** (FR-5.3): glyphs sorted queen→pawn with a material
  badge. **Rich summary panel:** per-side quality breakdown + "who played
  cleaner" verdict (human moves aren't engine-scored, said honestly).
- **Sounds** (off by default, persisted): CC0 piece sounds on move/capture/
  check/game-end. **Spectator reactions** (own off-by-default toggle): Pixabay
  crowd reactions keyed off the move-quality tag (`best`→applause,
  `blunder`→boo, `mistake`→disappointment, `inaccuracy`→shocked,
  checkmate→cheer), firing only on captures gated by a crowd-enthusiasm
  threshold (captured value + eval swing + quality premium vs a ply-decaying
  threshold — the crowd starts quiet and grows invested).

### 4.6 Tournament mode (LLM gauntlet) — FR-9.6
The setup screen's **Mode selector** (Single game / Tournament / Match —
§4.7) offers **Tournament** when exactly one side is LLM-AI; choosing it turns
the Start button into "Run tournament". (Replaces the earlier "Tournament
mode" checkbox; the selector also hosts Match mode. Tournament greys out the
Time control — it runs without a chess clock; Match honors it as a free
choice, default Off.) It plays a
**3-game round** between the LLM and the AI at each difficulty (1–11)
using an **adaptive gauntlet** (aligned with LLM-Chess's level-selection
heuristic): after each round, if the **AI swept** (LLM lost all 3) the
tournament **stops** — that level is the LLM's floor; otherwise (LLM swept
**or** mixed) it **climbs** to the next difficulty until the top level (11). (The earlier
"any 3-0 sweep ends it" rule ended a strong LLM at level 1 and never found its
ceiling; the adaptive rule is what makes the Elo estimate resolvable.)
Colors alternate each game; no chess clock (LLM not penalized for slow
inference / retry backoff); games capped at **200 half-moves → draw** (the
benchmark's 200-move cap) as a backstop; threefold repetition is also
detected (perpetual check draws at the 3rd repetition).

On completion a per-level results table plus diagnostics are shown and
auto-saved to `tournament-{model}-{yyyymmdd-hhmmss}.txt`:
- **Estimated Elo** (LLM-Chess-style MLE): maximum-likelihood rating over the
  (opponent Elo, game-score) pairs from the Stockfish `UCI_Elo`-anchored games
  (levels 6–10 only; 11 full and Normal 1–5 have no anchor). Solves
  `Σ(Sᵢ−Eᵢ(R̂))=0`, `Eᵢ(R)=1/(1+10^((Rᵢ−R)/400))` (bisection); 95% CI from
  Fisher info `I=ΣE(1−E)(ln10/400)²`, `SE=1/√I`. All-wins → "above {max
  anchor}", all-losses → "below {min anchor}", never-reached-Stockfish →
  "insufficient Elo-anchored data".
- **Instruction-following:** counts of LLM moves legal on first try / after
  retry / local fallback, as "% own legal moves".
- **Move-quality distribution** (best/good/inaccuracy/mistake/blunder %) and
  **game/efficiency stats** (games, avg moves/game, avg LLM latency/move
  excluding local analysis, avg completion tokens/move).
- **Methodology footnote:** our single-turn constrained-choice protocol is
  simpler than LLM-Chess's multi-turn agentic Game Duration, so our
  instruction-following % is not directly comparable; our Elo pool (Stockfish
  `UCI_Elo`) differs from theirs (Dragon/chess.com) though the MLE method is
  identical.

### 4.7 Match mode (best-of-N head-to-head) — FR-9.7
The Mode selector offers **Match** when **both sides are non-human** — the
headline use case is two LLM-AI sides (different models/endpoints, or the same
model with different personas to A/B-test system prompts), but a
Browser-AI/Stockfish side is allowed too. The user picks an **even game
count** (6 / 10 / 20 / 40, default 10); Start becomes "Run match". It plays
that many games between the two configured sides as-is — **no level sweep**
(that's the gauntlet's job, §4.6) — **colors alternating each game** so each
side plays White exactly N/2 times. Chess clock **optional when neither side is an LLM** (disabled if either side is an LLM, which isn't penalized for slow inference/retries); same 200-ply cap,
spectator pacing / Pause / Stop, and LLM retry/fallback as a normal game.

The output is a **relative** strength ranking (NOT absolute Elo — there's no
anchor, unlike the gauntlet's Stockfish-Elo MLE), shown and auto-saved to
`match-{labelA}-vs-{labelB}-{yyyymmdd-hhmmss}.txt`:
- **Result:** W-draw-L from side A's perspective (A = configured White, B =
  configured Black; identity fixed, colors alternate), plus each side's score.
- **Relative Elo difference:** `Δ = −400·log10(1/scoreA − 1)` (the standard
  logistic expected-score formula `E=1/(1+10^(−Δ/400))` used in engine testing
  — a *gap*, not a rating; not comparable to the gauntlet's anchored Elo or to
  FIDE). `scoreA` = wins + ½·draws over N. A shutout reports the gap as
  unbounded.
- **Likelihood of Superiority (LOS):** the Bayesian posterior `P(A truly
  stronger)` from decisive games only (draws carry no superiority signal),
  uniform prior → `Beta(W+1, L+1)`, `LOS = Σⱼ C(W+L+1, j)·0.5^(W+L+1)`; as a
  %, >95% conventional "significant". All-draws → "inconclusive".
- **Per-side diagnostics:** move-quality buckets (§4.5) for **both** sides
  (the match scores every non-human side, so a head-to-head is fair regardless
  of controller type — not LLM-only like the gauntlet); LLM sides additionally
  get IF% / avg latency / avg completion tokens. Side labels are
  `model [persona]` for LLMs (so same-model/different-prompt matches read
  clearly), else the engine name.
- **Methodology footnote:** relative (not absolute) Elo; LOS definition;
  single-turn constrained-choice; small N is noisy (N≥20 recommended).

**Why two modes:** the gauntlet (§4.6) answers *"how strong is this LLM?"*
(absolute Elo via the Stockfish ladder); Match answers *"which of these two is
stronger, and by how much?"* (relative Elo + LOS). The two share the game loop
behind a **series dispatch** layer (`seriesActive`/`seriesRecordResult`/…
route to whichever of `tournament`/`match` is running) so spectator pacing,
pause/stop, the 200-ply cap, and the results file are common to both.
Non-goal: absolute per-side Elo from a match (needs an anchor — use the
gauntlet).

---

## 5. Key contracts (don't break these)

- **ControllerConfig:** `{ type: 'human' | 'normal' | 'grandmaster' | 'llm',
  difficulty?, apiBase?, apiKey?, model?, systemPrompt? }` (spec §8, TDD §2).
  Internally, a grandmaster config also carries `elo` (from
  `GRANDMASTER_ELO_LEVELS`); it is not user-set.
- **Worker protocol** (spec §9, TDD §6):
  - Main→Worker: `{type:'search',...}`, `{type:'analyze', fen, uciMove}`
    (Normal only), `{type:'stop'}`.
  - Worker→Main: `{type:'result', move, evalCp, bestEvalCp}`,
    `{type:'analysis', evalCp, bestEvalCp}`, `{type:'error', message}`.
- **LLM request contract:** `POST {apiBase}/chat/completions`,
  `Authorization: Bearer {apiKey}` when a key is set, body `{ model, messages }`
  **only** (no temperature/top_p/etc. — §4.4). Response parsed for
  `choices[0].message.content` and `usage.completion_tokens`.
- **Eval convention:** scores are side-to-move-relative (negamax for Normal,
  UCI `score cp` for Stockfish), normalized to White's perspective before
  storage. Mate encoded as ±100000 ∓ N.
- **Move-quality tags** (PRD §5.4, TDD §4.5): five buckets by mover-perspective
  centipawn loss vs the best move — `best` ≤5, `good` ≤30, `inaccuracy` ≤90,
  `mistake` ≤200, `blunder` >200. Tunable baseline, not a contract.

---

## 6. Development standards

- **Single-file discipline.** Everything ships in `chess.html`. Do not add
  `.js`/`.css` files, build steps, or npm dependencies. The Stockfish runtime
  is vendored (the two sibling binaries, GPL-3.0); the only network fetch is
  the user-supplied LLM endpoint — nothing else.
- **Lazy by default.** Question whether a feature needs to exist. Reuse what's
  already in the file before writing new code. Stdlib/platform features before
  custom code. Shortest working diff wins — once you've read the whole flow.
  Mark deliberate simplifications with a `// ponytail:` comment naming the
  ceiling and the upgrade path.
- **Bug fixes target the root cause**, not the reported symptom. A guard in
  the shared function beats a guard in every caller.
- **Leave a check behind.** Non-trivial logic (a parser, a money/legality
  path, a branchy loop) gets the smallest runnable self-check — an
  `assert`-style `__main__`/demo block or one tiny test. Trivial one-liners
  need none.
- **Offline-first.** Human-vs-Human and Human-vs-Normal must work with no
  network. Only Grandmaster (Stockfish fetch) and LLM-AI (endpoint
  fetch) need the network.
- **Security at trust boundaries stays.** LLM API keys are sent directly from
  the browser to the endpoint. Per FR-9.4 the setup config (including the key)
  is persisted to `localStorage` so a restart proposes the last-used options
  on the same computer — the key is readable by anyone with this browser
  profile; acceptable for local use, not for sharing the file with a real key
  in it. Keys are never written into the HTML file itself.
- **Don't re-add LLM hyperparameters.** The request body is `{ model, messages
  }`. Adding `temperature` etc. breaks reasoning models and strict servers
  (§4.4). If steering is ever needed, add a *configurable* knob, never a
  hardcoded value.

---

## 7. Documentation discipline (this project's recurring failure mode)

This repo has three governing docs (spec/PRD/TDD) written *before* the code,
plus user-facing pages. They drift in both directions: docs under-describe
the code (a threshold or fallback lives only in code), and docs over-describe
the code (a feature is documented but not shipped). User-facing pages are the most dangerous
place for over-claiming.

**Rules:**
- When you change behavior in `chess.html`, update the matching spec/PRD/TDD
  section in the same commit.
- **Code is the source of truth.** If docs and code conflict, fix the docs
  unless the user explicitly wants the behavior to change.
- Never commit a user-facing page (`onefile-*.html`) that promises a feature
  the build lacks — cross-check against §9 first.
- When deferring a feature, record it in PRD §8.1 (deferred-features register)
  with a re-open trigger, and downgrade any spec FR it traces to.

---

## 8. Running & testing

- **Run:** open `chess.html` in a browser. Human-vs-Human and Human-vs-Normal
  work via `file://` with no server and no network. **Grandmaster (Stockfish)
  does NOT work via `file://`** — browsers block `new Worker()` from a
  `file://` page (same-origin policy on local files); the Normal engine only
  dodges this because it runs from an inline Blob URL. To use Grandmaster,
  serve the folder over HTTP (e.g. `python3 -m http.server`, then open
  `http://localhost:8000/chess.html`); it then runs fully offline from the
  vendored local bundle, no internet required. LLM-AI needs the network
  (the configured endpoint). LLM calls from an HTTPS page to an HTTP LAN
  endpoint are blocked as mixed content — serve the app over HTTP for local
  LLM servers.
- **Grandmaster / Stockfish:** the two-file bundle is **committed to the repo**
  next to `chess.html` (`stockfish-18-lite-single.js` ~20 KB loader,
  `stockfish-18-lite-single.wasm` ~7 MB; NNUE compiled in). The project is
  GPL-3.0, so vendoring the GPL Stockfish binary is consistent (spec §12.4).
  The one hard requirement is HTTP (not `file://`) serving.
- **Test:** there is **no in-repo test runner**. Validation done so far:
  perft-style rules-engine checks and Normal-engine search checks (run ad-hoc
  via `node` extracting the inline scripts), and Playwright/jsdom headless
  checks for LLM setup validation, retry-on-illegal, fallback-on-failure, and
  tournament paths. When you add non-trivial logic, keep the ad-hoc check
  runnable: extract the last `<script>` and `new Function()` it, or run a
  perft node count. A one-line `node -e` parse/self-check is the bar, not a
  test framework.

---

## 9. Feature status (shipped / deferred / future)

The authoritative deferred register is **PRD §8.1** (with re-open triggers);
this section is a compact map so agents don't re-derive it from a code scan.
Keep them in sync — change one, change both.

### 9.1 Shipped (built into `chess.html`)
Core: Human/Normal/Grandmaster/LLM controllers, eval bar, move-quality tags,
thinking indicator, AI-vs-AI spectating. Beyond the v1 baseline:
- **Tournament gauntlet (FR-9.6)** with adaptive climb, **LLM-Chess-style Elo
  estimate** (MLE), instruction-following / quality / efficiency diagnostics,
  methodology footnote, auto-saved results file.
- **Match mode (FR-9.7)** — best-of-N head-to-head between the two configured
  sides (LLM-vs-LLM, or LLM-vs-AI), giving a **relative** ranking: Elo
  *difference* (logistic from score) + LOS (Bayesian P(one side stronger)) +
  per-side diagnostics. Shares the game loop with the gauntlet behind a series
  dispatch layer. No absolute Elo (no anchor — that's the gauntlet).
- **LLM system-prompt selection** (5 personas + custom) and the
  **no-hyperparameter request body** (`{ model, messages }`).
- **PGN save/replay (FR-8.2)**, **confirm destructive actions (FR-6.5)**,
  **drag-and-drop (FR-5.2)**, **chess clock (FR-6.6)**, **threefold-repetition
  draw (FR-1.3)**.
- **Sampled sounds + spectator reactions** (CC0 / Pixabay, off by default),
  **pause/stop/speed spectator controls (FR-6.4)**, **captured-piece tray
  (FR-5.3)**, **rich summary panel**, **rematch**.
- **Plain-English move narration (FR-5.5)** under the board + **spoken TTS**:
  each move's narration (and the end-game verdict) can be spoken via a local
  OpenAI-compatible `/v1/audio/speech` endpoint (e.g. docker-kokoro / Kokoro)
  with a `speechSynthesis` fallback; vendor-neutral per §1.
- **Take over a side mid-game (FR-6.8)** — swap one side's controller
  (Human / Browser-AI / LLM-AI) from a modal and resume from the current
  position; single-game only (tournament/match end-and-continue is future).

### 9.2 Deferred (specified, not built — build when the trigger fires)
| Feature | Spec/PRD | Why deferred / re-open trigger |
|---|---|---|
| In-session AI-vs-AI result tally | PRD §4, §8 | Speculative UX on a working spectator flow. Re-open: a user asks for a session scoreboard. |
| Multi-threaded Stockfish WASM | spec A11.3, TDD §5.2 | Deployment host must serve COOP/COEP headers for `SharedArrayBuffer`. |
| Normal-engine Elo calibration | (new) | Levels 1–5 have no Elo anchor, so the tournament can't resolve models below Stockfish's 1350 floor (LLM-Chess's Dragon resolves to 250). Re-open: weak-model Elo resolution matters. |
| Accounts / server-side sharing | spec NG2 | NG2 bars automatic persistence; user-initiated PGN file I/O shipped (FR-8.2). Re-open: a concrete online-sharing need. |

### 9.3 Future ideas (need a product decision before building — none specced)
Each needs a spec FR + PRD/TDD section and a resolved open question first.
- **LLM single hint** (human-requested): needs its own LLM config (humans have
  no endpoint today) + a definition of the hint budget.
- **Per-side language (EN/IT):** scope unclear — per-side *UI* language (two
  players, one screen?) vs commentary vs announcements. No in-app strings are
  localized today.
- **AI commentary per move** (LLM-generated, not the local narration): the
  *spoken-narration* half of the old "commentary + TTS" item shipped (FR-5.5 —
  move narration spoken via local Kokoro / `speechSynthesis` fallback). What
  remains is **LLM-generated commentary** on each move (vs the local
  describeMove sentence): needs a dedicated commentary endpoint/profile, a
  latency/pacing plan that doesn't block the loop or AI-vs-AI pacing, and a
  profile/language selector.
- **Configurable LLM reasoning level:** `reasoning_effort`/thinking-budget
  knob. The param mapping is endpoint/model-specific (needs a capability
  probe); note the benchmark found more thinking ≠ better chess (Claude 3.7
  extended thinking gave only +17% and even degraded).
- **NoN / MoA multi-model orchestration:** pair a smart-but-erratic reasoning
  model with a cheap non-reasoning orchestrator to fix instruction-following
  (LLM-Chess lifted R1 32%→63%, Gemini 42%→79% this way). Out of scope for our
  single-LLM-per-side architecture today.
- **Per-turn hard time cap** on top of the chess clock, if a "no single move
  may exceed N seconds" guard is ever wanted.
- **Take over a side in a series:** the single-game take-over (FR-6.8) is
  disabled in tournament/match because a series re-sets `matchConfig` every
  game and its stats/Elo assume fixed matchups. End-and-continue would need to
  record the swap and treat prior games as the old matchup's — a deliberate
  change to the series result/Elo model.

**Lowest-friction next build:** among §9.3, the **LLM single hint** is the
highest-value but needs a config decision first. Everything here
carries a real product question — pick one and talk through the decision
before building.
