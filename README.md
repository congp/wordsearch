# Word Search

A mobile-first word search game. Tap a letter and swipe across a word to find it. Found words get a neon highlight, and the highlight colours follow a set of rules so that crossing words stay easy to tell apart. The page is plain static HTML with no build step or dependencies, Live at **<https://congph.com/wordsearch/>**, hosted on GitHub Pages from [github.com/congp/wordsearch](https://github.com/congp/wordsearch).

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

Run that from the repository root, then open <http://localhost:8000/>. The same server is set up in `.claude/launch.json` as "Word Search (local)", for tools that start previews from that file.

## Files

| File | What it is |
|---|---|
| `index.html` | The whole game: styles, markup and one script inside an IIFE (a function that runs immediately). |
| `words-info.js` | The paragraph shown for every word, grouped by puzzle, as `window.WORD_INFO`. |
| `docs/how-it-works.md` | Developer guide: architecture, rules, algorithms, saved data, testing. |
| `docs/design.md` | Design guide: tokens, layout, components, states, motion, sound, copy. |
| `docs/analytics.md` | Analytics: what is tracked (Umami, cookie-free), the events, and the questions they answer. |
| `AGENTS.md` | Brief for AI coding agents: product decisions to keep, how to check changes. |
| `CLAUDE.md` | Points Claude Code at `AGENTS.md`. |
| `.claude/launch.json` | Local preview server config. |

## Privacy

The game uses [Umami](https://umami.is), a cookie-free analytics service, to count visits (with approximate location: country, region and city) and gameplay events such as which puzzles are started and finished. It doesn't use cookies or collect names, emails or other personal data, and it doesn't send players' saved best times. Details are in `docs/analytics.md`. A footnote at the bottom of the game says the same.

To stop your own visits being counted, open `congph.com/wordsearch/?analytics=off` once in each browser you use (`?analytics=on` undoes it).

## Hosting and deploying

- **Host:** GitHub Pages, deploying from the `main` branch, repository root. There's no build step: the files are served as they are.
- **Address:** the repo is named `wordsearch`, and the owner's GitHub Pages site uses the custom domain `congph.com`, so this project is served at `congph.com/wordsearch/`. Renaming the repo changes the address.
- **Deploying:** **every push to `main` goes live** within a minute or two. Check changes locally before pushing.
- **Everything is public,** these docs included. GitHub Pages serves Markdown files as pages, for example `/wordsearch/docs/design.html`.
- **Players' saved data** (settings and best times) is stored per web address. Moving the game to a different domain or path starts every player fresh.
