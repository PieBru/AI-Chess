# AGENTS.md — Granpy Piero's AI Chess

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
2. **Evaluate AI "intelligence."** The app is also a lens for comparing the
   engines — how Normal behaves at each difficulty, how strong Grandmaster is,
   and concretely how each thinks move-to-move. This goal drives the
   transparency features (eval bar, move-quality tags, thinking indicator,
   AI-vs-AI spectating) that a plain "let me play chess" app wouldn't need.

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
specs/                        # Project / governing documentation (SDD layers)
  chess-app-spec.md           # SDD Layer 1 — spec (requirements, the contract)
  chess-app-prd.md            # SDD Layer 2 — PRD (UX, flows, behavior)
  chess-app-tdd.md            # SDD Layer 3 — TDD (architecture, design, schemas)
docs/                         # User-facing pages (blog / guide / kids, EN + IT)
  onefile-chess-*.html
```

No `package.json`, no build tooling, no test runner — by design. The project
is **GPL-3.0** and **vendors** the Stockfish runtime (the two sibling files
above) so Grandmaster works out of the box, offline, including on the live
site (spec §12.4). See §6 and §8.

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
  (`stockfish-18-lite-single.{js,wasm}`) and loaded as a local Worker (CDN
  fallback kept). Single-threaded
  on purpose: no COOP/COEP/`SharedArrayBuffer` requirement, so the file works
  via `file://` or any static host. Multi-threading is a deferred upgrade
  (spec A11.3 / §13, TDD §5.2).
- **Hand-rolled 0x88 rules engine** (`Rules` namespace) — legal move
  generation, FEN/SAN, check/checkmate/stalemate, draws (threefold, fifty-move,
  insufficient material). `chess.js` is the documented fallback (spec §12.2)
  only if perft tests fail.
- **`fetch()` to an OpenAI-compatible chat endpoint** for LLM-Assisted mode,
  straight from the main thread. No SDK, no proxy.

---

## 4. Architecture (chess.html)

The file is one big `<script>` after the markup. Top-down:

### 4.1 Rules engine (`#rules-src`)
Pure functions, no DOM/worker dependency. Owns the 0x88 board, move
generation, FEN/SAN, game-end detection. Key surface: `Rules.newGame`,
`genLegalMoves`, `applyMove`, `toFEN`/`fromFEN`, `toSAN`, `gameStatus`,
`zobristHash`, `sqName`/`nameToSq`. This is reused inside the worker, so it
lives in the inline `#rules-src` block that gets concatenated into the worker
blob.

### 4.2 Normal engine (`#normal-engine-src`)
Hand-built alpha-beta with null-move pruning, late-move reductions, a capped
transposition table, and per-difficulty move-selection noise. Difficulty table
(DIFFICULTY, ~line 1012):

| Level | maxDepth | timeMs | noise |
|---|---|---|---|
| 1 Beginner | 1 | 50 | wide-random |
| 2 Easy | 3 | 150 | top3-random |
| 3 Medium | 4 | 500 | gaussian-small |
| 4 Hard | 6 | 1500 | gaussian-tiny |
| 5 Expert | 99 | 4000 | none |

(TDD §4.3 is the contract; constants are tunable.) Runs in a worker and
answers two message types: `search` (pick a move) and `analyze` (grade an
arbitrary played move — used for quality tagging of *any* move source).

### 4.3 Grandmaster engine
Stockfish over UCI-in-a-worker: `uci`→`uciok`→`isready`→`readyok` handshake,
then `position fen … moves …` + `go movetime 3000` per move, parsing
`score cp`/`score mate` and `bestmove`. Loaded lazily; load failure disables
the option for the session (PRD §5.5).

### 4.4 LLM-Assisted controller
Not a worker — a main-thread `fetch` to `{apiBase}/chat/completions`. Prompt
sends FEN + side + SAN history + the **full legal UCI move list** (constrained
choice = the main reliability lever). One corrective retry feeding the bad
reply back into the conversation; on final failure, **falls back to a local
Normal move (difficulty 3)** so play continues (LLM APIs are transiently
flaky; a flaky endpoint must not kill a game). Evals for the bar/tags come
from the Normal engine's `analyze` message — the chat endpoint returns none.

### 4.5 Game loop & UI
`playTurn()` is the spine: checks game status, dispatches to the side's
controller via the engine-agnostic `requestAIMove()` (all tiers resolve to
`{ move, evalCp, bestEvalCp }`), re-validates every AI move through the rules
engine before committing (NFR-7.2), then renders. AI-vs-AI games auto-play by
`playTurn()` recursing at the loop tail. Setup screen captures per-side
`ControllerConfig`; `matchConfig` is the live `{white, black}` pair.

---

## 5. Key contracts (don't break these)

- **ControllerConfig:** `{ type: 'human' | 'normal' | 'grandmaster' | 'llm',
  difficulty?, apiBase?, apiKey?, model? }` (spec §8, TDD §2).
- **Worker protocol** (spec §9, TDD §6):
  - Main→Worker: `{type:'search',...}`, `{type:'analyze', fen, uciMove}`
    (Normal only), `{type:'stop'}`.
  - Worker→Main: `{type:'result', move, evalCp, bestEvalCp}`,
    `{type:'analysis', evalCp, bestEvalCp}`, `{type:'error', message}`.
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
  the user-supplied LLM endpoint (plus a Stockfish CDN backstop if the local
  bundle is ever absent) — nothing else.
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
  network. Only Grandmaster (Stockfish fetch) and LLM-Assisted (endpoint
  fetch) need the network.
- **Security at trust boundaries stays.** LLM API keys are sent directly
  from the browser to the endpoint. Per FR-9.4 the setup config (including
  the key) is persisted to `localStorage` so a restart proposes the last-used
  options as defaults on the same computer — so the key is readable by anyone
  with this browser profile; acceptable for local use, not for sharing the
  file with a real key in it. Keys are never written into the HTML file itself.

---

## 7. Documentation discipline (this project's recurring failure mode)

This repo has three governing docs (spec/PRD/TDD) written *before* the code,
plus user-facing pages. They drift. Two drift directions, both have bitten us:

1. **Docs under-describe the code** (e.g. move-quality thresholds lived only in
   `classifyQuality`; LLM failure-fallback was documented as "error out").
2. **Docs over-describe the code** — the bigger current gap: six documented
   features the build does not ship (Appendix A). User-facing pages are the
   most dangerous place for this.

**Rules:**
- When you change behavior in `chess.html`, update the matching spec/PRD/TDD
  section in the same commit.
- **Code is the source of truth.** If docs and code conflict, fix the docs
  unless the user explicitly wants the behavior to change.
- Never commit a user-facing page (`onefile-*.html`) that promises a feature
  the build lacks — cross-check against Appendix A first.
- When deferring a feature, record it in PRD §8.1 (deferred-features register)
  with a re-open trigger, and downgrade any spec FR it traces to.

---

## 8. Running & testing

- **Run:** open `chess.html` in a browser (or `firefox chess.html`). No server
  required; works via `file://`. Human-vs-Human and Human-vs-Normal need no
  network. LLM-Assisted needs the network (the configured endpoint).
- **Grandmaster / Stockfish:** the two-file Stockfish bundle is **committed
  to the repo** next to `chess.html`:
  - `stockfish-18-lite-single.js` (~20 KB loader)
  - `stockfish-18-lite-single.wasm` (~7 MB; NNUE compiled in)
  The project is GPL-3.0, so vendoring the GPL Stockfish binary is consistent
  (spec §12.4). Grandmaster therefore works **out of the box — fully offline
  and on the live site** — with no extra download. The loader tries the local
  bundle first, then a CDN fallback; the CDN backstop is currently broken
  (jsDelivr refuses the >150 MB npm package with HTTP 403), but the committed
  local bundle makes that irrelevant.
- **Test:** there is **no in-repo test runner**. Validation done so far:
  - Perft-style rules-engine checks and Normal-engine search checks (run
    ad-hoc, e.g. via `node` extracting the inline scripts).
  - jsdom headless tests for LLM setup validation, retry-on-illegal, and
    fallback-on-failure paths (run externally by the maintainer).
- When you add non-trivial logic, keep the ad-hoc check runnable: extract the
  last `<script>` and `new Function()` it, or run a perft node count. A
  one-line `node -e` parse/self-check is the bar, not a test framework.

---

## Appendix A — (formerly) deferred features — now shipped

> **Status (2026-07-05): all six items below are implemented in `chess.html`.**
> This appendix previously tracked documented-but-unshipped features; they are
> now live, so the doc-vs-code gap it warned about is closed. The spec FRs
> (FR-5.3, FR-6.4, FR-7.2) and PRD sections (§2.3, §2.4, §5.4, §6) were never
> downgraded, so no standing needed restoring — only this notice and the
> PRD §8.1 sound row were updated.

| # | Feature | Doc source | Implementation note |
|---|---|---|---|
| 1 | **Pause/Stop AI-vs-AI** | spec FR-6.4, PRD §2.3 | Spectator-only control bar. Pause halts the loop after the in-flight move; Stop ends the game with reason "stopped by spectator" (result `*`). Generation tracking prevents a move computed for a superseded game (e.g. via Rematch) from leaking into a new one. |
| 2 | **Spectator speed control** (Normal/Fast/Instant) | PRD §2.3 | Select inserts a 700/200/0 ms pause between AI-vs-AI moves; persisted to `localStorage`. |
| 3 | **"Rematch" action** (same config) | PRD §2.4 | Same controllers, straight into a new game — no setup-screen detour. |
| 4 | **Rich summary panel** | PRD §2.4 / §5.4 | Per-side quality breakdown (best/good/inaccuracy/mistake/blunder + average centipawn loss) and a "who played cleaner" verdict. Human moves are not engine-scored, so the panel says so honestly rather than treating them as perfect. |
| 5 | **Captured-piece tray** | spec FR-5.3 | Glyphs sorted queen→pawn with a small material-advantage badge. |
| 6 | **Sound effects + off-by-default toggle** | PRD §6, §8 | Synthesized via Web Audio (no external assets — closes PRD §8.1's "assets deferred" open item). Off by default; toggle persists locally. |

**Judgment calls where docs were silent:**
- Pause/Stop/Speed render only in true AI-vs-AI spectating (both sides
  non-human), not in mixed human/AI games.
- Stop records result `*` (not a win/loss), per the spectator-aborts-game case.
- "Who played cleaner" is decided by average centipawn loss, with an honest
  caveat when one or both sides lack engine data (human moves).

### What remains unbuilt (two tiers)

With Appendix A closed, exactly two categories of unbuilt work remain. The
registers named here are the **source of truth**; this section is a map so
agents don't re-derive it from a code scan. Do not duplicate the rows into
AGENTS.md — link to the register, or it will drift (§7).

**Tier 1 — Deferred-but-specified (in PRD §8.1).** Specced as FRs/PRD
features, product decisions made, deliberately not coded. Build when the
re-open trigger fires; no new design questions to resolve first.

| Feature | Spec/PRD ref | Re-open trigger |
|---|---|---|
| In-session AI-vs-AI result tally | PRD §4, §8 | A user asks for a session scoreboard |
| PGN export / accounts / sharing | PRD §4, spec NG2 | v1.1 candidate; a concrete export need surfaces (relaxes NG2) |
| Drag-and-drop move input | spec FR-5.2, A11.4 | Confirmed before locking UX post-launch (code is click-to-move only today) |
| Multi-threaded Stockfish WASM | spec A11.3, TDD §5.2 | Deployment host confirmed to serve COOP/COEP headers |

**Tier 2 — Future ideas needing decisions (Appendix B below).** Not specced,
not promised, each carries open product questions. Build none of these until
the listed decisions are made and a spec FR + PRD/TDD section exists. Quick
map of the open blockers (full detail in Appendix B):

1. **Save/load/replay (PGN)** — reopens spec NG2/FR-8.1; needs a storage target.
2. **LLM single hint** — needs its own LLM config (humans have no endpoint
   today) + a definition of the hint budget.
3. **Turn timer** — adds a new game-over reason (time forfeit); fairness
   differs per controller type.
4. **Per-side language (EN/IT)** — **most open UX**: per-side *UI* language
   (two players, one screen?) vs commentary vs announcements.
5. **AI commentary + multilingual TTS** — which LLM, which TTS engine,
   latency/pacing, profile selector. Audio foundation now exists (Appendix A #6).
6. **Configurable LLM reasoning level** — param mapping is
   endpoint/model-specific; needs a capability probe; interacts with #3.
7. **Confirm destructive actions** (New game / Resign) — confirm-only piece
   ships now with zero open questions; the save-on-confirm branch gates on #1.
8. **Move-evaluation SFX** — sound-source tradeoff (synthesized vs sampled);
   can share audio plumbing with #5.

**Lowest-friction next builds:** Tier 1 drag-and-drop is fully specified;
Tier 2 #7 (confirm dialogs) is the only future idea with no open product
question and ships in ~5 lines.

---

## Appendix B — Future feature ideas (not started; for planning only)

Recorded so they aren't forgotten. None are specced, designed, or promised to
users. Each would need a spec FR + PRD/TDD section before implementation.
See Tier 2 above for the quick map of open decisions per item.

1. **Save / load / replay a game session** (completed or in-progress),
   including **replay from step N**.
   - *Hook:* `moveHistory` already records every ply (SAN + move + evals) —
     serialization is straightforward; replay = rebuild state from `START_FEN`
     by applying moves 1..N, then park the game loop. PGN is the natural format
     (already a v1.1 candidate in PRD §8.1).
   - *Decision needed:* this **reopens spec NG2 / FR-8.1** (no cross-session
     persistence is currently a hard non-goal). Pick a storage target
     (`localStorage`, file download/upload) and explicitly relax NG2.

2. **LLM-assisted single hint** requestable by a human player, with a
   **budget capped by the chosen game level**.
   - *Hook:* reuse the LLM `fetch` path with a "suggest, don't move" prompt;
   render as a square/arrow highlight, never an auto-move.
   - *Decisions needed:* (a) humans have no difficulty setting today — define
   what "game level" gates the hint budget (opponent's difficulty? a fixed
   per-game counter?); (b) in Human-vs-Normal/Grandmaster there is no LLM
   endpoint configured, so the hint needs its own LLM config independent of the
   opponent's controller.

3. **Turn timer**, configurable, **applying to AI and LLM players too** — so a
   strong-but-slow engine/LLM can't simply out-think a weaker-but-faster one.
   - *Hook:* wrap `requestAIMove()` in `Promise.race` with a wall-clock timeout;
   on expiry either forfeit the move or force a quick fallback move.
   - *Decision needed:* this adds a **new game-over reason (time forfeit)** —
   `finishGame` and the result banner gain a loss-on-time case. Also note
   engines already have internal budgets (Normal `timeMs`, Stockfish
   `movetime`, LLM = unbounded network latency); the turn timer is a
   wall-clock cap *above* those, with different fairness implications per
   controller type.

4. **Per-opponent language selection** (English / Italian, extensible) — each
   side picks its own language.
   - *Hook:* needs an in-app i18n string table (the `onefile-*_it.html` pages
   prove EN/IT translations exist for marketing copy, but **no in-app strings
   are localized today** — the UI is single-language via `<html lang>`).
   - *Decision needed:* clarify scope — is this per-side **UI** language (two
   players sharing one screen in different languages?), per-side **commentary
   / thinking text**, or move announcements? The sharing-one-screen case has
   real layout/contrast implications and should be pinned down before any code.

5. **AI commentary per move, spoken via multilingual TTS.** After each move,
   an LLM generates spoken commentary for the current context (move, SAN,
   eval, quality tag), played through a multilingual TTS engine (e.g. Piper,
   QwenTTS, or similar). Pure entertainment — opponents and spectators *hear*
   the game. Configurable **language/profile** (serious, soccer-style,
   satiric, humorous, …).
   - *Hook:* reuse the LLM `fetch` path with a commentary prompt; feed the
   returned text to TTS. The **lazy/zero-dependency first option is the
     browser-native `SpeechSynthesis` API** (Web Speech) — multilingual, no
     network, no dependency; reach for Piper/QwenTTS (server or WASM) only if
     native voices fall short.
   - *Decisions needed:* (a) which LLM — reuse a side's LLM config, or a
     dedicated commentary endpoint? (b) TTS engine choice (native vs. hosted);
     (c) profile/language selector UI; (d) **latency** — commentary per move
     adds wall-clock delay and must not block the game loop or AI-vs-AI pacing
     (generate async, queue/decay if moves come faster than speech); (e) the
     audio-output foundation now exists — Appendix A #6 shipped a synthesized
     Web Audio path with an off-by-default toggle, which TTS commentary can
     build on.

6. **Configurable LLM reasoning/thinking level**, from none up to the max the
   model allows (e.g. reasoning-effort / thinking-budget knobs, or a
   chain-of-thought prompt prefix). Lets a side trade latency/cost for move
   quality — directly serving the "evaluate AI intelligence" goal (how much
   does extra reasoning actually help an LLM play chess?).
   - *Hook:* add a per-side `reasoning` field to `ControllerConfig`; map it to
     either an OpenAI-style `reasoning_effort`/`max_completion_tokens` request
     parameter (when the endpoint supports it) or a prompt-side
     think-then-move scaffold (when it doesn't). Reuse the existing
     `apiBase`/`apiKey`/`model` config.
   - *Decisions needed:* (a) the param mapping is **endpoint/model-specific** —
     not every OpenAI-compatible server exposes the same knob, so this needs a
     capability probe or a per-model setting; (b) prompt-scaffold fallback is
     weaker and must still return one legal UCI move (don't break the
     constrained-choice contract); (c) higher reasoning = more latency/cost,
     which interacts with Appendix B #3 (turn timer) — a reasoning-capped LLM
     under a timer is a distinct fairness scenario worth defining together.

7. **Confirm destructive actions** (New game, Resign), with an option to save
   the current game on confirm. Today both buttons are immediate and
   non-reversible — a stray click/keystroke ends or discards an in-progress
   game. Add a confirmation step; if a game is in progress, offer to save it
   first (ties into Appendix B #1 save/load).
   - *Hook:* interpose a confirm dialog before `finishGame` (Resign) and
     before returning to the setup screen (New game); the save-offer branch
     reuses B #1's serialization once it exists.
   - *Decisions needed:* (a) confirmation UX — native `confirm()` is the
     zero-dependency lazy default; an inline modal is nicer but more code;
     (b) the save-offer only makes sense once B #1 lands, so this splits into
     a standalone confirm-now piece and a save-on-confirm piece gated on B #1.

8. **Move-evaluation SFX** — play a sound keyed off the move-quality tag
   (e.g. applause for `best`, a "Boooo" for `blunder`). Pure entertainment,
   reinforcing the quality tags audibly for players and spectators.
   - *Hook:* `classifyQuality` already returns the tag per move — trigger a
     cue from the same point tags are rendered. Off-by-default toggle per
     PRD §6 / §8 (Appendix A #6).
   - *Decisions needed:* the **sound-source tradeoff** is the real question:
     (a) **synthesized via native `AudioContext`** (Web Audio) — zero-
     dependency, fits the single-file stance, but synthesized applause/boo is
     limited in expressiveness; (b) **sampled clips** (real applause/boo) are
     expressive but either bloat the file as base64 or require a network fetch,
     both against the single-file/no-fetch grain. Likely answer: synthesize
     simple tones for the milder tags and accept limited expressiveness, or
     treat sampled SFX as the one exception that fetches from a CDN (like
     Stockfish). The sound foundation (Appendix A #6) is now built, so this
     can share the audio-output plumbing with B #5 (TTS commentary).

When one of these is chosen for implementation: move it out of this appendix,
write the spec FR + PRD/TDD sections, and add any new non-goal relaxations to
the §4 / NG lists explicitly.
