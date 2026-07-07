# Piero's AI Chess, 2026 Edition — Product Requirements Document (PRD)

**Document type:** PRD (SDD Layer 2 — user-facing behavior, flows, UX)
**Derived from:** `chess-app-spec.md`
**Status:** Draft v1.0

---

## 1. Product goals

- **Primary goal — entertain.** The app should be fun to play against and fun to watch, regardless of skill level. A beginner should be able to have an enjoyable, winnable game; a strong player should be able to get a genuine challenge.
- **Secondary goal — evaluate AI "intelligence."** The app is also a lens for comparing the two AI tiers: how strong is Normal at each difficulty, how strong is Grandmaster, and — concretely — how do they actually think and differ move to move? This shapes several requirements below that a pure "just let me play chess" app wouldn't need (transparency into engine thinking, move-quality feedback, AI-vs-AI spectating).

These two goals are complementary, not competing: transparency features that satisfy the evaluation goal (eval bar, move-quality tags, thinking stats) also make the game more engaging to watch, which serves entertainment.

## 2. Primary user flows

### 2.1 Setup flow
1. User lands on a **Setup screen**: two side-by-side controller pickers, one per color (White / Black).
2. Each picker offers: Human, Browser-AI (→ reveals a Difficulty sub-picker, 1–11 per spec FR-3.2 — **levels 6–11 are Grandmaster / Stockfish**, so Grandmaster is no longer a separate top-level choice), LLM-AI (→ reveals Endpoint / API key (optional) / Model fields per spec FR-9.1, independent per side).
3. A short one-line description accompanies each option so a non-technical user understands what they're choosing (see §5.1 copy).
4. "Start Game" begins play; the board screen loads with both sides' controllers locked for that game (spec FR-2.3).
5. An optional **Time control** selector (Off / Bullet 1+0 / Blitz 3+2 / Rapid 10+5 / Classical 30+0, default Off) arms a FIDE-style chess clock (spec FR-6.6): each side gets a total time budget plus a per-move increment. When on, two live clocks appear in the side panel; the side-to-move's clock ticks down (shown in red under 10 s) and a clock hitting zero ends the game as a loss on time. Applies in every mode, but engines rarely flag — in practice it constrains humans and slow LLMs.
6. An optional **Move limit** selector (Off / 40 / 60 / 80 / 100 / 150 / 200 full moves, default Off, spec FR-1.4) caps a single game's length: reaching the limit ends it as a draw — a safety net against very long engine games (the fifty-move rule only fires after 100 *consecutive* quiet plies, which engines can dodge with periodic captures/pawn moves). Off = rely on the fifty-move rule. Shown in Single and Match mode (in Match it can only tighten the 200-half-move safety net); hidden in Tournament, which keeps its fixed cap. Persisted per browser.

### 2.2 Gameplay flow (Human involved)
1. On a human turn: legal-move highlighting on piece selection, click-to-move **and drag-and-drop**; an illegal click reselects/deselects and an invalid drop snaps back — no modal, no dead end.
2. On an AI turn: board is read-only, a **thinking indicator** appears (see §5.3) — this is not just a spinner, it's part of the "evaluate the AI" experience.
3. After every ply: move appended to history panel in SAN, last-move highlighted, check indicator if applicable.
4. Game end: a summary panel appears (see §2.4).

### 2.3 Spectator flow (AI vs AI)
This flow exists specifically to serve the secondary goal — watching two AIs (or two difficulty levels of the same AI) play each other is the most direct way to "evaluate how smart each one is."
1. User configures both sides as AI (any tier/difficulty combination, including Normal vs Normal at different difficulties, or Normal vs Grandmaster).
2. A **playback speed control** (Very slow / Slow / Normal / Fast / Instant) governs how long the app pauses between moves purely for watchability — the AI still computes at its real budget; this only affects the pacing of what's shown.
3. Pause/Resume and Stop controls are always visible (spec FR-6.4).
4. The eval bar and move-quality tags (§5.4) are especially prominent in this mode, since there's no human move to focus on.

### 2.4 Game-end / summary flow
1. Result banner: winner + reason (checkmate, resignation, stalemate, draw — with the specific draw reason per spec FR-1.3).
2. **Game summary panel**, addressing the secondary goal directly:
   - Move-quality breakdown per side (count of Best / Good / Inaccuracy / Mistake / Blunder — see §5.4 for definition).
   - For AI vs AI games specifically: a plain-language one-line takeaway, e.g. "White (Grandmaster) outplayed Black (Normal, Level 3) — 0 mistakes vs 4."
   - "Rematch" (same config) and "New setup" actions.

## 3. Feature requirements (mapped to spec)

| PRD feature | Spec ref | Notes |
|---|---|---|
| Controller picker (per side) | FR-2 | Human / Normal(+difficulty 1–11, where 6–11 = Grandmaster) / LLM-AI |
| Grandmaster engine asset | FR-4, spec §12.1 | Stockfish WASM (`nmrugg/stockfish.js`, single-threaded) fetched from CDN |
| Rules engine | FR-1, spec §12.2 | Hand-rolled 0x88 in v1; `chess.js` retained as documented fallback |
| Board + move input | FR-1, FR-5 | Click-to-move + drag-and-drop |
| Move history (SAN) | FR-5.4 | Scrollable, always visible |
| Thinking indicator | NFR-3 | Must reflect worker activity, never fake/static |
| Eval bar | new — see §5.4 | Not in spec v1; added here to serve secondary goal |
| Move-quality tags | new — see §5.4 | Same |
| AI vs AI spectator controls | FR-6.4 | Speed control is new, added here |
| Game summary panel | FR-7 | Extended beyond spec's minimum result line |
| Resign / Flip / New Game | FR-6 | |
| Graceful Stockfish-load failure | FR-4.3 | Must not silently fail — see §5.5 |

## 4. Non-goals for this PRD

Per spec §3, and additionally for this PRD specifically:
- No onboarding tutorial or rules explainer — the audience is assumed to already know how to play chess (the AI-comparison angle implies a somewhat engaged, curious user).
- No persistent cross-session stats (spec NG2). Within a single browser session, a lightweight running tally of AI-vs-AI results *could* be kept in memory purely for the spectator flow (resets on reload — explicitly transient, not a database), but is **deferred past v1** per the §8 decision below.
- No user accounts, sharing, or export (e.g. exporting a PGN) in v1 — flagged as a good v1.1 candidate given the evaluation goal, but not required now.

## 5. UX requirements by component

### 5.1 Setup screen copy
Each controller option needs a plain-language strength cue so the "evaluate intelligence" goal starts before the game even begins:
- Normal — Beginner: *"Makes mistakes on purpose. Good for learning."*
- Normal — Easy: *"Plays casually — will punish obvious blunders but gives chances."*
- Normal — Medium: *"A solid club-level challenge."*
- Normal — Hard: *"A strong club player. Few free gifts."*
- Normal — Expert: *"The strongest this hand-built engine can play."*
- Grandmaster: *"Stockfish. Full strength. This one doesn't lose on purpose."*
- LLM-AI: *"A chat model picks each move from the legal options. Point it at any OpenAI-compatible endpoint — fun for seeing how badly an LLM plays chess.""

### 5.2 Board & interaction
- Standard 8×8 board, light/dark square theme, coordinate labels on the edge.
- Selected piece highlighted; legal destinations marked distinctly from captures (capture squares visually different from empty destination squares).
- Check: king square gets a distinct warning highlight (not just a text notice — should be readable at a glance while watching a fast AI-vs-AI game).

### 5.3 Thinking indicator (entertainment + evaluation)
While an AI is searching, show, live if possible:
- Which engine is thinking (Normal L3 / Grandmaster).
- A lightweight progress cue tied to iterative deepening — e.g. current depth reached — for the Normal engine, and elapsed think time for Grandmaster. This is a direct, low-effort window into "how the AI thinks," serving the secondary goal without requiring a dedicated analysis screen.
- This must never block or lag the UI (NFR-3) — it's a display of worker progress messages, not a computation on the main thread.

### 5.4 Evaluation bar & move-quality tags (new feature, serving secondary goal)
- A slim vertical or horizontal bar showing the current position's evaluation (from White's perspective), sourced from whichever engine most recently searched the position. If only a human just moved and no engine has evaluated the resulting position yet, the bar shows the last known value with a subtle "pending" state until an engine (even a quick background Normal-engine call) scores it.
- After each move, classify it by comparing the played move's resulting eval to the best available move's eval (both numbers already produced by search, so this is cheap):
  - **Best** — matches or ties the top move.
  - **Good** — small eval loss.
  - **Inaccuracy / Mistake / Blunder** — increasing eval loss thresholds.
- Tags render as small badges next to the relevant move in the history panel. This is the single most direct product mechanism for "how smart was that move" — it turns an abstract engine-strength question into a readable, per-move label.
- Centipawn thresholds for each tag are recorded as a tunable baseline in TDD §4.5 (≤5 best, ≤30 good, ≤90 inaccuracy, ≤200 mistake, >200 blunder, mover-perspective loss); expect a calibration pass against real games (spec A11.1, §8.1 register).

### 5.5 Error & edge states
- Grandmaster selection when Stockfish fails to load: the option is disabled with an inline explanation ("Grandmaster mode needs an internet connection to load Stockfish") rather than allowed-then-failing mid-game (spec FR-4.3).
- Worker crash mid-game: non-blocking toast/banner ("The AI ran into a problem — start a new game to continue") plus a New Game shortcut; the board freezes in place rather than corrupting (spec NFR-7.1).

## 6. Visual & tonal direction

Since the primary goal is entertainment, the app should not read as a bare technical demo:
- Warm, inviting board theme by default (not sterile gray/white); should still meet contrast requirements (spec NFR-6.2).
- Small delight touches are encouraged and left to design exploration: move animations (piece sliding, not teleporting), a subtle capture effect, a distinct and satisfying checkmate moment (e.g. brief highlight/animation on the mating position).
- Sound effects (move, capture, check, game-end) — optional, user-toggleable, off by default is a reasonable safe choice pending design input.
- The evaluation features (§5.4) should feel like a natural, integrated part of the experience — closer to a broadcast-style eval bar than a debugging panel.

## 7. Success signals (qualitative, v1 has no analytics/backend to measure these quantitatively)

- A user can explain, after one AI-vs-AI game, roughly how the two engines differed in strength — this is the core signal that the secondary goal is met.
- A beginner can complete and enjoy a game against Normal/Beginner without feeling like they need to read documentation first.
- Nothing in the flow requires leaving the page (aside from the one-time Stockfish asset fetch) or reading this PRD.

## 8. Open questions for design/implementation

- Exact move-quality thresholds (§5.4) and difficulty-to-ELO mapping (spec A11.1) — needs playtesting, not a UX decision. **Not blocking v1**; tuned post-launch against real games.
- Dependency choices (Stockfish build, rules-engine build-vs-buy) are now settled in spec §12 — no longer UX-open; the only UX-facing artifact is the §5.5 Stockfish-load-failure copy, which the chosen single-threaded build makes *more* robust (no COOP/COEP prerequisite to fail on). **Closed.**
- ~~Whether the in-session AI-vs-AI tally (§4) is worth building for v1 or deferred~~ — **Resolved (2026-07-04): defer past v1.** Zero spec/TDD hooks (FR-6.4 pause/stop and FR-7.2 result line are independent of it); it is speculative UX layered on a working spectator flow. The §4 carve-out stays open so the door isn't foreclosed; build it (~15 lines: an in-memory counter bumped on each game result, rendered in the summary panel) only if the spectator flow earns it.
- Sound on/off default and asset choice — design call, not blocking. **Resolved (2026-07-04): off by default, user-toggleable (§6 line 108 already states this as the safe choice); assets chosen at implementation time. No UX question left open for v1.**

### 8.1 Deferred features (consolidated register)

Single source of truth for anything pushed past v1. Each item keeps the door open and names the re-open trigger.

| Feature | Source | Decision | Re-open trigger |
|---|---|---|---|
| In-session AI-vs-AI result tally | PRD §4, §8 | Defer past v1 | Spectator flow ships and a user asks for a session scoreboard |
| ~~PGN export / accounts / sharing~~ | PRD §4, spec NG2 | **Shipped (2026-07-05):** user-initiated PGN export (download) + import (view-only replay via First/Prev/Next/Last). Not accounts/sharing and not automatic persistence — NG2/FR-8.1 narrowed to permit file I/O only. | Accounts/ratings or server-side sharing, if a real sharing need surfaces |
| ~~Drag-and-drop move input~~ | spec FR-5.2, A11.4 | **Shipped (2026-07-05):** click + drag share one resolver (`attemptMoveTo`); invalid drops snap back | n/a |
| ~~Sound effects default + assets~~ | PRD §6, spec A11.5 | **Shipped (2026-07-05):** off-by-default toggle, persisted. Piece sounds are **sampled CC0** (lichess `standard`/`sfx` themes — move/capture/check/game-end), embedded as base64 data URIs (~31 KB raw) so the file stays self-contained and offline. (Earlier synthesized version was replaced after playtest feedback.) | n/a |
| ~~Spectator reactions~~ (Appendix B #8) | PRD §6 | **Shipped (2026-07-05), upgraded to sampled same day:** off-by-default "Spectator reactions" toggle, persisted. Real crowd reactions under the **Pixabay License** (no attribution, GPL-3.0-compatible): `best`→ applause, `blunder`→ boo, `mistake`→ disappointment, `inaccuracy`→ shocked, checkmate→ cheer. **Reactions fire only on captures, gated by a crowd-enthusiasm threshold:** each move gets a drama score (captured-piece value + eval swing in pawns + quality premium) and fires only if it beats a linear threshold that decays with ply count — the crowd starts quiet (early game, only queen/rook captures or big swings fire) and grows more invested (by the endgame, any capture gets a reaction). This avoids the old problem of constant applause in strong matches while simulating a real audience's growing engagement. Embedded as base64 in `#reaction-clips-src` (~171 KB raw / ~228 KB base64, 5 clips). Earlier synthesized Web Audio version (noise-burst "claps") sounded like static — replaced. | n/a |
| ~~Chess clock / turn timer~~ (Appendix B #3) | spec FR-6.6 | **Shipped (2026-07-05):** optional FIDE-style per-side total clock + per-move increment (Off / Bullet 1+0 / Blitz 3+2 / Rapid 10+5 / Classical 30+0, default Off, persisted). The side-to-move's clock runs during its move (human deliberation or engine think); the spectator pacing delay does NOT count. A clock reaching zero = loss on time (new game-over reason). Chose the faithful total-clock model over a simple per-turn cap (a per-turn cap isn't how chess works). ponytail simplification: no "opponent has insufficient mating material → draw" arbiter rule; simple loss on time. | Insufficient-material-on-flagfall draw rule, or a per-side custom (minutes + increment) input |
| Multi-threaded Stockfish WASM | spec A11.3, §12; TDD §5.2 | v1.1 upgrade, header-gated | Deployment host confirmed to serve COOP/COEP |
| ~~Full offline Grandmaster mode~~ | spec NG3 | **Done** — the Stockfish bundle is vendored in the repo (GPL-3.0, NFR-5.2), so Grandmaster runs fully offline once served over HTTP (no internet download); `file://` is not supported for GM because browsers block its Worker (NFR-5.2) | n/a |
| Move-quality tag thresholds + ELO mapping | PRD §5.4, spec A11.1 | Not blocking; tuned post-launch | Playtest data available |
| ~~Threefold-repetition draw detection~~ | spec FR-1.3 | **Shipped (2026-07-07):** the 3rd occurrence of the same position (key = FEN board + side to move + castling + ep, ignoring the move clocks) → draw ("threefold repetition"). Catches perpetual check / shuffle loops early (vs the fifty-move rule's ~100-ply wait); the position count is seeded with the start position and reset per game. Tournament/match games also keep the 200-half-move cap (FR-9.6/FR-9.7) as a backstop for loops that never quite repeat. | n/a |

None of the above blocks v1 scope (spec FR/AC set, PRD §5 UX). All are additive and reversible.
