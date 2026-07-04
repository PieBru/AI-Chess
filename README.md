# Granpy Piero's AI Chess

▶️ **Play it now:** <https://piebru.github.io/AI-Chess/chess.html>

📖 **Read more:**
- Blog — 🇬🇧 [English](https://piebru.github.io/AI-Chess/docs/onefile-chess-blog.html) · 🇮🇹 [Italiano](https://piebru.github.io/AI-Chess/docs/onefile-chess-blog_it.html)
- Player's guide — 🇬🇧 [English](https://piebru.github.io/AI-Chess/docs/onefile-chess-guide.html) · 🇮🇹 [Italiano](https://piebru.github.io/AI-Chess/docs/onefile-chess-guide_it.html)
- For kids — 🇬🇧 [English](https://piebru.github.io/AI-Chess/docs/onefile-chess-kids.html) · 🇮🇹 [Italiano](https://piebru.github.io/AI-Chess/docs/onefile-chess-kids_it.html)

A single-file chess app built with vanilla HTML, CSS, and JavaScript — no build step, no frameworks, no npm. `chess.html` **is the whole app**: open it and play. The only other files in the repo are the Stockfish engine binaries, which are **optional** and used **only** for Grandmaster mode — every other mode (Human, Normal AI, LLM-Assisted) runs entirely from that one file.

I made it to have fun while entertaining my grandkids on both playing and designing such software, and to have an empirical benchmark platform to measure my self-hosted systems' quality about LLM inferencing, harnesses, scaffolds, software engineering, and coding. Two goals, in priority order: **entertain** (a beginner should have a winnable, enjoyable game; a strong player should get a genuine challenge) and **evaluate AI "intelligence"** — concretely, how the engines and LLMs think move-to-move.

NOTE: it's almost impossible for me to manually test all features and options. EXPECT BUGS and please open an issue if you find one, or want to suggest new features.

## Features

- **Single-file app** — everything lives in one `chess.html`: no build step, no frameworks, no CDN for core play. (The Stockfish files are an optional companion for Grandmaster mode only — see [Play](#play).)
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

Open `chess.html` in any modern browser (or `firefox chess.html`). It works via `file://` — no server required. **`chess.html` alone is a complete game**: Human vs Human, Human vs Normal AI, and LLM-Assisted all run from that one file with nothing else.

**Grandmaster mode is the one exception** — it needs the Stockfish engine, shipped as two optional companion files next to `chess.html` (`stockfish-18-lite-single.{js,wasm}`, GPL-3.0). They load only when a side is set to Grandmaster; delete them and every other mode keeps working unchanged.

## Specs

Full design documentation (Spec-Driven Development):
- [`specs/chess-app-spec.md`](specs/chess-app-spec.md) — requirements, the contract
- [`specs/chess-app-prd.md`](specs/chess-app-prd.md) — UX, flows, behavior
- [`specs/chess-app-tdd.md`](specs/chess-app-tdd.md) — architecture, design, schemas
- [`AGENTS.md`](AGENTS.md) — operating manual for the project

User-facing pages (blog, player's guide, kids): see [`docs/`](docs/).

## Development environment

This project was developed fully locally on **Arch Linux**, driven by the [**pi** coding agent](https://pi.dev) backed by a self-hosted **llama.cpp** server running **Qwen-3.6-27B_UD-Q8 by Unsloth** — no cloud APIs involved. (Part of the point: an empirical benchmark for self-hosted LLM inferencing and agentic software engineering.)

## License

Licensed under the [GNU General Public License v3.0](LICENSE).

This project vendors the **Stockfish** runtime (`stockfish-18-lite-single.{js,wasm}`, GPL-3.0) directly in the repository, so the whole project is GPL-3.0 — consistent copyleft across the app and the bundled engine. Those files are an optional companion (Grandmaster mode only); `chess.html` itself is a complete single-file app without them.
