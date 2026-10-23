# Word Search

A mobile-first word search game. Tap a letter and swipe across a word to find it. Found words get a neon highlight, and the highlight colours follow a set of rules so that crossing words stay easy to tell apart. The page is plain static HTML with no build step or dependencies, served from this GitHub Pages site at `/wordsearch/`.

## Features at a glance

- **10 puzzles:** Animals, Fruits, Ocean, Space, Sports, Weather, and four The Office (US) puzzles. Each board is a fresh 10×10 grid with 20 words drawn from that puzzle's word list.
- **Lots of crossings:** the generator deliberately packs words so they overlap, like a printed puzzle book. Words can even hide inside other words (SKI inside SKIING).
- **Neon highlights:** six colours in a fixed order (yellow, green, orange, blue, purple, pink). The colour switches by rule, and overlapping highlights mix into new shades.
- **Game modes:**
  - **In order:** words must be found alphabetically.
  - **Crazy:** a countdown (10–60 s) that refills each time you find a word. If it runs out, the board resets.
- **Word cards:** tap any word to read a short paragraph about it.
- **Records:** completion count and personal best time per puzzle, kept separately for each mode combination.
- **Generative music:** an ambient soundtrack created live in the browser, which reacts to progress and the crazy-mode clock. It is off by default.

## Run it locally

```bash
python3 -m http.server 8000
```

Run that from the repository root, then open <http://localhost:8000/wordsearch/>. The repository's `.claude/launch.json` has the same server configured as "Static File Server (Python)".

## Files

| File | What it is |
|---|---|
| `index.html` | The whole game: styles, markup and one script inside an IIFE (a function that runs immediately). |
| `words-info.js` | The paragraph shown for every word, grouped by puzzle, as `window.WORD_INFO`. |
| `docs/how-it-works.md` | Developer guide: architecture, rules, algorithms, saved data, testing. |
| `docs/design.md` | Design guide: tokens, layout, components, states, motion, sound, copy. |
| `AGENTS.md` | Brief for AI coding agents: product decisions to keep, how to check changes. |

## Publishing note

GitHub Pages publishes everything in this folder, these docs included: Markdown files are served as pages at `/wordsearch/docs/…`.
