# Notes for AI agents working on Word Search

This is a single-page, mobile-first word search game. Read `README.md` first, then the guide for your task:

- **Logic, features, data, testing:** `docs/how-it-works.md`
- **UI, tokens, components, states, copy:** `docs/design.md`

## Ground rules

- **No build step and no dependencies.** It's plain HTML, CSS and JS in `index.html`, plus the `words-info.js` data file. Don't add frameworks, bundlers or packages without asking the owner.
- **Match the existing code style:** an IIFE, `const`/`let`, small named functions, short comments that explain *why*. Everything stays inside the IIFE; nothing becomes global except `window.WORD_INFO`.
- **Wrap every `localStorage` access in `try/catch`.** The game must work with no storage at all.
- **Pushing to `main` deploys to the live site** (congph.com/wordsearch/) through GitHub Pages. Don't push or merge to `main` without the owner's OK.
- **Analytics events are a contract.** The names and fields listed in `docs/analytics.md` feed the owner's Umami dashboards; don't rename or remove them. New player actions that matter get a `track()` call and a row in that doc. Never send personal data or anything from `localStorage`.
- **Don't break saved data.** If you change any `wordsearch-*` key or the stats JSON shape, migrate the old data.

## Product decisions to keep

The owner made these deliberately. Ask before changing any of them; the reasons are in `docs/how-it-works.md` §16.

1. **Board:** a 10×10 grid with 20 words per board, placed with **maximum overlap**.
2. **Nesting:** words hidden inside other words (SKI in SKIING) are allowed.
3. **Highlight colours:** six neon colours in a fixed order, yellow → green → orange → blue → purple → pink. Colour **Rules 1–4** decide switching (`colorAllowed`); keep their behaviour exactly.
4. **Overlap:** means the highlight *bars* touch, tested geometrically in `overlaps()`. The bar width (`stroke-width` 0.7) must stay **below 0.707** or the test stops matching what players see.
5. **Highlight look:** `opacity: 0.5` with `mix-blend-mode: multiply` on a **light** board, so overlaps mix colours.
6. **Order:** the word list and the Puzzles list are alphabetical.
7. **Crazy mode:** the clock starts on first touch, a found word refills it, and time-up gives a **new board in the same puzzle**.
8. **Pausing:** both clocks pause while any `<dialog>` is open or the tab is hidden. Use the existing `suspend()`/`resume()`; don't add your own pause logic.
9. **Personal bests:** kept per mode combination, and a mode counts only if it was on for the **whole** game.
10. **Music:** generated live with Web Audio, no audio files, off until the player taps the button.
11. **Fit:** the game screen fits 375 × 812 with no scrolling during play.

## When you add or change content

- **New puzzle:** its word list in `THEMES` **and** its name in exactly one category in `CATEGORIES`.
- **New word:** A–Z, 2–10 letters, added to `THEMES`, **and** a paragraph added to `words-info.js` under the same puzzle name. Whenever `words-info.js` changes, bump the `?v=` on its `<script>` tag in `index.html`, or returning players get stale descriptions.
- **Paragraphs:** factual, one or two sentences. For The Office, never invent episode details; if unsure, keep it general or ask.

## Verify before you report done

There's no test suite, so check in a browser at 375 × 812 (recipes in `docs/how-it-works.md` §14):

1. **Errors:** the page loads with no console errors.
2. **Solve:** solving a board with the console snippet ends in the "🎉 Done in …" status.
3. **Fit:** puzzles you touched still place 20 words (Surnames is about 18).
4. **Colours:** if you touched colours, overlap or generation, colour Rules 2–4 hold, checked with an independent overlap test.
5. **Layout:** no horizontal scroll (`scrollWidth === 375`), and the page still fits vertically.
6. **Timing:** if you touched timing, the clocks pause while a dialog is open (use real waits).
7. **Cache:** add a cache-busting query (`?v=N`) when reloading, because the local server caches pages.

Report what you verified and what you couldn't. For example, you can't *hear* audio in an automated browser, so measure its levels instead and say so.
