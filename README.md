# Granpy Piero's AI Chess

A single-file chess app built with vanilla HTML, CSS, and JavaScript — no build step, no frameworks, no npm. Just open the file and play.

I made it to have fun while entertaining my grandkids on both playing and designing such software, and to have an empirical benchmark platform to measure my self-hosted systems' quality about LLM inferencing, harnesses, scaffolds, software engineering, and coding. Two goals, in priority order: **entertain** (a beginner should have a winnable, enjoyable game; a strong player should get a genuine challenge) and **evaluate AI "intelligence"** — concretely, how the engines and LLMs think move-to-move.

NOTE: it's almost impossible for me to manually test all features and options. EXPECT BUGS and please open an issue if you find one, or want to suggest new features.

## Features

- **Self-contained** — everything in one `chess.html`, no build step, no frameworks, no CDN for core play.
- **Hand-rolled 0x88 rules engine** — legal move generation, FEN/SAN, check/checkmate/stalemate, and draws (threefold, fifty-move, insufficient material). No `chess.js` dependency.
- **Four opponent types, pickable per side:**
  - **Human** — local two-player on one board.
  - **Normal** — a hand-built alpha-beta search (null-move pruning, late-move reductions, transposition table) with five difficulty levels, from wide-random Beginner to full-depth Expert.
  - **Grandmaster** — Stockfish over UCI, the strongest engine there is.
  - **LLM-Assisted** — any OpenAI-compatible endpoint *you* configure; the model is given the full legal move list (constrained choice) with a retry on an illegal reply and a graceful local fallback so a flaky endpoint never kills a game.
- **Mix any two sides** — Human vs Normal, Stockfish vs LLM, or sit back and **spectate AI vs AI**.
- **Engine transparency** — eval bar, per-move quality tags (`best` / `good` / `inaccuracy` / `mistake` / `blunder`), and a "thinking…" indicator, so you can *see* how each side reasons.
- **Web Workers** — the Normal engine and Stockfish run off the main thread, so the UI never blocks.
- **Offline-first** — Human vs Human and Human vs Normal need no network. Grandmaster loads a local Stockfish runtime bundle when present (else tries a CDN); LLM-Assisted needs your configured endpoint.
- **Setup persistence** — your per-side controller choices and LLM config are saved to `localStorage` and proposed as defaults next time.

## Play

Open `chess.html` in any modern browser (or `firefox chess.html`). It works via `file://` — no server required.

For **offline Grandmaster** strength, drop the two-file Stockfish bundle next to `chess.html`:
- `stockfish-18-lite-single.js` (~20 KB loader)
- `stockfish-18-lite-single.wasm` (~7 MB; NNUE compiled in)

Download them from the [`nmrugg/stockfish.js`](https://github.com/nmrugg/stockfish.js) GitHub release (v18.0.0). They are GPLv3 and intentionally **not** committed to this repo.

## Specs

Full design documentation (Spec-Driven Development):
- [`specs/chess-app-spec.md`](specs/chess-app-spec.md) — requirements, the contract
- [`specs/chess-app-prd.md`](specs/chess-app-prd.md) — UX, flows, behavior
- [`specs/chess-app-tdd.md`](specs/chess-app-tdd.md) — architecture, design, schemas
- [`AGENTS.md`](AGENTS.md) — operating manual for the project

User-facing pages (blog, player's guide, kids): see [`docs/`](docs/).

## License

Licensed under the [Apache License, Version 2.0](LICENSE).

The **Stockfish** runtime used optionally by Grandmaster mode is **GPL-3.0**. It is loaded strictly at arm's length as a separate Web Worker speaking the UCI text protocol and is never bundled or vendored into this repository, so its copyleft does not reach into the app's own (Apache-2.0) source.
