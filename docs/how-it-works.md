# How the game works

This is the developer guide to the Word Search game: what each part does, the rules it follows, and why. Everything below lives in `index.html` (one IIFE, so nothing is global) and `words-info.js`. Function and constant names are given so you can search for them; line numbers are left out because they drift.

## Contents

1. [Architecture](#1-architecture)
2. [Game state](#2-game-state)
3. [Puzzles and word lists](#3-puzzles-and-word-lists)
4. [Board generation](#4-board-generation)
5. [Selecting a word (input)](#5-selecting-a-word-input)
6. [Highlight colours and their rules](#6-highlight-colours-and-their-rules)
7. [Game modes](#7-game-modes)
8. [Pausing: suspend and resume](#8-pausing-suspend-and-resume)
9. [Completion records and personal bests](#9-completion-records-and-personal-bests)
10. [Word cards](#10-word-cards)
11. [Music and sound effects](#11-music-and-sound-effects)
12. [Saved data (localStorage)](#12-saved-data-localstorage)
13. [Common changes](#13-common-changes)
14. [Testing and verifying](#14-testing-and-verifying)
15. [Known limitations](#15-known-limitations)
16. [Decision log](#16-decision-log)
17. [Analytics](#17-analytics)

---

## 1. Architecture

- **One static page, no build step.** `index.html` holds the CSS in `<style>`, the markup, and one `<script>` containing an IIFE. `words-info.js` loads first and defines `window.WORD_INFO`.
- **No frameworks or dependencies.** The game uses the DOM, SVG, Web Audio, `localStorage`, Pointer Events and `<dialog>` directly.
- **How it draws:**
  - **Letters:** a CSS grid of `<div class="cell">` elements inside `#grid`.
  - **Highlights:** an SVG overlay, `#overlay`, sits *behind* the letters. Its `viewBox` is `0 0 SIZE SIZE`, so **one SVG unit equals one cell**. Each highlight is a `<line>` from the centre of the first letter to the centre of the last, with round end caps.
  - **Selection:** while swiping, a separate light-blue line, `selectLine` (`.pill-select`), follows the finger.

Code sections, in order, inside the IIFE:

| Section | Main names |
|---|---|
| Config and data | `SIZE`, `WORDS_PER_GAME`, `THEMES`, `DIRS`, `COLORS` |
| Music engine | `Sound` (a small module with its own private state) |
| Board generation | `tryPlace`, `generate`, `newGame` |
| Music button | `setSound`, the pointerdown "kick" |
| Crazy mode clock | `renderClock`, `tick`, `pauseClock`, `startClock`, `armClock`, `timeUp` |
| Pausing | `suspend`, `resume` |
| Records | `watchStart`, `watchPause`, `watchReset`, `recordWin`, `loadStats`, `fmtTime` |
| In order mode | `nextWord`, `markNext`, `rejectOutOfOrder` |
| Drawing helpers | `makeLine`, `setLine` |
| Input geometry | `cellAt`, `snapEnd`, `pathBetween` |
| Colour rules | `overlaps`, `collinear`, `colorAllowed`, `pickColor` |
| Input handlers | `pointerdown`, `pointermove`, `endSelection` |
| Puzzle list | `renderList` |
| Word cards | the click handler on `#words` |

## 2. Game state

These are top-level variables inside the IIFE.

| Variable | Meaning |
|---|---|
| `theme` | Name of the current puzzle; a key of `THEMES`. |
| `grid` | `SIZE × SIZE` array of single letters. |
| `words` | The words on this board, **sorted alphabetically**. |
| `found` | Array of found-word entries (shape below), in the order they were found. |
| `colorIdx` | Index in `COLORS` of the *current* highlight colour. |
| `start`, `current` | `{r, c}` of the swipe's first and current cell, or `null` when not swiping. |
| `gameId` | Increases on every `newGame()`; used to ignore stale pause state from an earlier board. |
| `deadline`, `remaining`, `raf`, `resetting` | Crazy-mode clock state (§7.2). |
| `playMs`, `playFrom`, `playModes` | Stopwatch state for records (§9). |

A **found-word entry** looks like this:

```js
{ word: "TIGER", from: {r, c}, to: {r, c}, dir: [dr, dc], color: 0..5 }
```

`from` and `to` are the cells where the swipe started and ended, so `from` may be the word's last letter if it was swiped backwards. `dir` is the unit step from `from` to `to`.

## 3. Puzzles and word lists

`THEMES` maps each puzzle name to its word list. Each game draws `WORDS_PER_GAME` (20) words from the list at random.

**Rules for words:**
- **Letters:** A–Z only, 2 to `SIZE` (10) letters, upper case.
- **Duplicates:** no duplicates within a puzzle.
- **List size:** keep at least about 30 words per puzzle so replays feel different. Long words lower how many fit (see the note on "The Office: Surnames" in §15).
- **Words inside other words are allowed and intended.** SKI/SKIING, SUN/SUNNY, DART/DARTS. The generator will usually nest the short word inside the long one (§4), and Rule 3 (§6) gives the two highlights different colours.
- **Every word needs a paragraph** in `words-info.js` under the same puzzle name (§10).
- **Accented words lose their accents in the grid** (PHỞ → PHO, ÁO DÀI → AODAI, gemütlich → GEMUTLICH), and multi-word names are joined. The paragraph gives the proper spelling.
- **Keep the average word length around 6 letters.** Puzzles of long words can't fit 20 on a 10×10 grid; add short entries to compensate.

**Categories.** `CATEGORIES` is an ordered list of `{ name, puzzles: [...] }`. Every puzzle in `THEMES` must appear in exactly one category; any that don't show under "Other". `categoryOf(name)` looks a puzzle's category up. There are 8 categories: Pop Culture, Food & Drink, Art & Lifestyle, Science & Discovery, Nature, Travel, Play & Adventure, Mind & Language. Category names are sent to analytics, so don't rename them casually.

**The Puzzles list** (`renderList`):
- **Layout:** one section per category, in `CATEGORIES` order. Each has a heading and an "N of M completed" count, with its puzzles sorted alphabetically.
- **Filter:** a sideways-scrolling row of chips at the top: "All", then one per category, each with its puzzle count. The chosen filter is saved in `wordsearch-list-filter`.
- **Cards:** each card folds its word list behind a "N words" toggle (`<details>`).
- **Opening the list:** it scrolls to the current puzzle.

The word chips under the grid are alphabetical because `generate()` sorts `words`.

## 4. Board generation

`generate(themeName)` returns `{ grid, words }`.

1. **Draw words:** shuffle the puzzle's list, take the first 20, and sort them longest first. Long words are hardest to fit, so they go in while the grid is empty.
2. **Place each word** with `tryPlace(grid, word)`:
   - It tries **every** start cell in **all 8 directions** (`DIRS`, which includes backwards and both diagonals).
   - A spot is valid if the word fits inside the grid and every cell is either empty or already holds the same letter.
   - Among valid spots it picks one with the **most shared letters**, with ties broken at random. This greedy "maximum overlap" choice is what packs 20 words onto 100 cells and creates the crossings. A spot that sits entirely on another word's letters (full nesting) counts as the maximum overlap, so nesting happens whenever it can.
3. **Swap in replacements:** if a word didn't fit, try the remaining shuffled words from the puzzle until there are 20.
4. **Retry:** repeat the whole attempt up to 40 times and keep the attempt that placed the most words. It usually succeeds on the first attempt.
5. **Fill the rest:** empty cells get random letters A–Z.

Measured with 20–100 simulated boards per puzzle:
- most puzzles place 20 words every time;
- Untranslatable Words, Name That Feeling and Inside Your Brain average 19.7–19.9;
- "The Office: Surnames" averages about 18, because its names are long.

A short word can also appear by chance in the random filler letters. Swiping that copy still counts as finding the word.

## 5. Selecting a word (input)

The whole `.board` has `touch-action: none`, so swiping never scrolls the page. Input uses Pointer Events, so mouse, touch and pen all work.

- **`pointerdown`:**
  - Ignored if the board is solved or is resetting.
  - Otherwise it starts the crazy clock and the stopwatch (§7.2, §9), captures the pointer, and sets `start = current = cellAt(e)`.
- **`pointermove`:** sets `current = snapEnd(start, cellAt(e))` and redraws the selection line.
  - **`cellAt`** turns pointer coordinates into a cell, clamped to the grid. Dragging outside the board still works.
  - **`snapEnd`** snaps the drag to the **nearest of the 8 directions** by angle, then shortens it until the end stays inside the grid. A wobbly finger still selects a clean straight line.
- **`pointerup` / `pointercancel` → `endSelection()`:**
  1. Read the letters along the path (`pathBetween`) as a string `s`.
  2. A **match** is any *unfound* word equal to `s` or to `s` reversed. Words can be swiped in either direction.
  3. If In order mode is on and the match isn't `nextWord()`, reject it (§7.1).
  4. Otherwise create the found-word entry, choose its colour (§6), and draw the highlight line. The line is inserted before `selectLine` so it sits underneath it.
  5. Update the chip: add the `found` class and set its background to the highlight colour.
  6. Then:
     - refill the crazy clock;
     - vibrate (30 ms);
     - pass progress to the music;
     - show the status, or run the win flow (§9).

## 6. Highlight colours and their rules

`COLORS` is a fixed, ordered list of six neon colours: yellow `#f2ff00`, green `#39ff14`, orange `#ff9500`, blue `#00d5ff`, purple `#c04dff`, pink `#ff2ec8`.

These rules were specified by the product owner, so keep their behaviour exactly.

| # | Rule | Implementation |
|---|---|---|
| 1 | Start with one colour, yellow, and **stay on the current colour** until a rule forces a switch. A switch moves to the next colour in the list and *stays* there, wrapping from pink back to yellow. | `colorIdx` persists between words; `pickColor` tries `colorIdx`, then `colorIdx + 1`, and so on. |
| 2 | A colour may overlap itself **only once per word**. Each highlight may be crossed by at most one other highlight of its own colour. | `colorAllowed`: reject if the new word would cross 2 or more same-colour words, or if the one same-colour word it crosses already has a same-colour crossing. |
| 3 | If two words **share letters and lie on the same straight line** (one nested in the other, or overlapping end to end), the new word must use a different colour. | Reject the colour if any same-colour overlapping word is `collinear`, meaning a parallel or opposite `dir`. Collinear and overlapping together means the same line. |
| 4 | A highlight may cross several colours, but **if it crosses 2 or more highlights and one of them is the current colour, switch**. | Reject the colour if `overlapping.length >= 2` and any of them has that colour. |

Sometimes **no colour can satisfy the rules**, for example when a long word crosses highlights of all six colours. That happened in 7 of about 6,000 highlights in testing. `pickColor` then chooses the least-bad colour: it never shares a colour with a same-line word if it can avoid it, then crosses as few same-colour words as possible, with ties going in the usual order.

Rule 4 is stricter than Rule 2. Rule 2 alone would allow a yellow word to cross one yellow and one green word; Rule 4 does not. The product owner chose to keep both.

### What "overlap" means

Two highlights overlap when their **bars visibly touch**. That covers sharing a letter, and also two diagonals crossing in an X *between* letters, where they share no cell. `overlaps(a, b)` treats each bar as the segment from `from` to `to` (cell centres) and runs a standard segment-intersection test (`orient` and `onSeg`, including collinear overlap).

This matches what you see only because of the bar width. Bars are drawn with `stroke-width: 0.7` cells. The closest two non-intersecting bars on this grid ever get is 1/√2 ≈ 0.707 cells (neighbouring parallel diagonals). At 0.7 they never touch. At the old width of 0.78 they slightly touched without counting as an overlap. **If you change the stroke width, the overlap test no longer matches what players see.** Keep it below 0.707, or switch to a distance-based test.

### How overlaps look

Found highlights use `opacity: 0.5` and `mix-blend-mode: multiply`, so crossings mix like paint (yellow over blue looks green). Multiply needs a light background behind the board; see `design.md`.

## 7. Game modes

Both switches are in the "Game modes" card. They can be used together, and each saves its own setting (§12).

### 7.1 In order

- **Only the next word counts:** only the first unfound word in alphabetical order (`nextWord()`) can be found. Its chip gets the `.next` outline (`markNext`).
- **Out-of-order swipes:** swiping a different real word is rejected (`rejectOutOfOrder`):
  - the chip shakes (`.nope`);
  - the status reads "Not yet — find X first";
  - the "nope" sound plays and the phone vibrates.
  - The crazy clock is **not** refilled.
- **Switching it mid-game** works and takes effect immediately.

### 7.2 Crazy

- **The countdown:** set by the `#crazySecs` select (10, 15, 20, 30, 45 or 60 seconds; default 30).
- **Clock states:**
  - **Armed:** a full clock that isn't running. A new board is armed (`armClock`).
  - **Running:** the clock starts on the **first touch** of the board, so the player gets a moment to look first. `deadline` is set and a `requestAnimationFrame` loop calls `tick()`.
  - **Refill:** finding a word resets `remaining` to the full time.
  - **Urgent:** when 5 seconds or less are left, the bar and the "⏱ Ns" text turn red, and the music ticks and brightens (§11).
  - **Time up** (`timeUp`):
    - set `resetting = true`, which blocks input;
    - shake the board with a red ring (`.timeup`);
    - vibrate and play the time-up sound;
    - after 1.1 seconds, `newGame(theme)` builds a **new board in the same puzzle** (not the same grid, so players can't just memorise positions);
    - the clock goes back to armed.
  - **Stopped:** the clock stops once the board is solved.
- **Paused:** whenever a dialog is open or the tab is hidden (§8).
- **Settings changes:** turning Crazy on or off, or changing the seconds, re-arms the clock.

## 8. Pausing: suspend and resume

`suspend()` pauses both clocks, the crazy countdown and the records stopwatch, and remembers which were running *for this `gameId`*. `resume()` restarts them only when **no `<dialog>` is open and the tab is visible**.

- **What pauses:** opening the Puzzles list, opening a word card, and the page being hidden all call `suspend()`. The dialogs' `close` events and the page becoming visible again call `resume()`.
- **Why it checks for open dialogs:** a dialog's `close` event fires asynchronously. Without the check, closing one dialog could restart the clocks while another was still open. That bug was found and fixed during testing.
- **Why it checks `gameId`:** if a new board started while paused (for example, by choosing a puzzle from the list), the old pause state belongs to a different game and is ignored. The new board's clock stays armed until the first touch.

## 9. Completion records and personal bests

- **Stopwatch:**
  - It starts on the first swipe (`watchStart`) and builds up time in `playMs`.
  - It pauses via `suspend()`, and resets on every `newGame()`, including crazy-mode resets.
- **Mode snapshot:** on the first swipe, `playModes = { order, crazy }` records the modes that are on. Switching a mode **off** during the game clears it from the snapshot. Switching one **on** mid-game doesn't add it. **A mode only counts if it was on for the whole game.**
- **Mode key:** `modeKey(playModes)` gives `normal`, `order`, `crazy` or `order+crazy`. `MODE_NAMES` holds the labels shown to players.
- **On win** (`recordWin`): increase `done`, and store the time as the best for that mode key if it's faster or the first. It returns `{ ms, key, prev, isBest }`. The status then shows one of:
  - "🎉 Done in 2:14 — first finish!"
  - "🎉 Done in 1:58 — new personal best!"
  - "🎉 Done in 2:30 · best 1:58"

  A mode suffix such as "(In order)" is added when the mode isn't normal.
- **Display:** `fmtTime` shows `m:ss`, or `h:mm:ss` past an hour. Comparisons use exact milliseconds.
- **Puzzles list:** completed puzzles show "✓ Completed ×N" and "Best:" with one time for each mode key recorded.

Stats schema, stored in `localStorage["wordsearch-stats"]` as JSON:

```json
{
  "Animals": { "done": 3, "best": { "normal": 133000, "crazy": 160500 } }
}
```

## 10. Word cards

- **Opening a card:** tapping any word chip, found or not, opens the `#infoDlg` dialog. It shows:
  - the puzzle name;
  - the word;
  - `WORD_INFO[theme][word]`, or "No description yet for this word." if there's none;
  - a colour band, which is the word's highlight colour if found, or neutral otherwise.
- **Clocks:** both clocks pause while it's open.
- **Data:** `words-info.js` is keyed by **puzzle, then word**, because the same word can mean different things in different puzzles (RAY in Space vs Ocean, CAT, SEAL, SUN, SURF, KARATE).
- **Writing style:** one or two plain, factual sentences, about 15–40 words. The Office entries describe characters and episodes. Don't invent episode details; if you're unsure, keep it general.

## 11. Music and sound effects

`Sound` is a self-contained module. Everything is synthesised live with Web Audio; **there are no audio files**. The approach was inspired by 1in8b.life, but the chords, voices and timing are original.

**Signal chain:** notes → `bus` → lowpass filter (1700 Hz, or 3200 Hz when urgent) → dry (0.6) + convolver reverb (0.5 wet) → `master` gain (`VOLUME` = 2.2, fades in over 1.5 s and out over 0.6 s) → compressor (−18 dB threshold, 3:1 ratio) → speakers. The reverb's impulse response is 3.2 seconds of stereo noise fading by `(1 − t)³`.

**Note voice** (`note`): a sine or triangle oscillator with a small random detune (±4 cents), a linear attack and an exponential fade, plus an optional quiet sine an octave up for warmth.

**Music loop** (`musicStep`, every 4 s via `setInterval`):
- **Chords:** on every second step (every 8 s), the next chord from `CHORDS` plays as a slowly rolled swell (2.2 s attack, 9.5 s fade). The loop is Fmaj7 · Em7 · Dm7 · Cmaj7 · Am7 · Dm9 · Gsus · G, about 64 seconds in all.
- **Melody:** on every step, 0 to `most` random notes from the C major pentatonic scale (`PENTA`) at random times within the 4 s window.
  - `most` is `1 + round(energy × 3)`, plus 2 when urgent.
  - Once energy is above 0.6, it uses the upper part of the scale.

**Game inputs:**

| Call | Effect |
|---|---|
| `setEnergy(found / words)` | More of the board found → more melody notes, and higher. |
| `setUrgent(bool)` | Crazy clock in its last 5 s → filter opens to 3200 Hz, more notes. |
| `found(i)` | Rising three-note chime, one scale step higher per word. |
| `nope()` | Two low falling notes (out-of-order swipe). |
| `tick()` | Soft high blip, once per second while urgent. |
| `timeUp()` | Four falling notes. |
| `win()` | Run up the scale, then a big soft C chord. |

**Behaviour:**
- **Off by default:** browsers only allow audio after a user gesture, so `setOn(true)` must be called from a tap.
- **Saved "on":** if the player turned music on last time (`wordsearch-music = "1"`), the button shows as on and the music starts on the first `pointerdown` anywhere on the page.
- **Hidden tab:** the audio context is suspended while the tab is hidden.
- **iPhone silent switch:** the code sets `navigator.audioSession.type = "playback"` (iOS Safari 17 and later) so sound plays with the silent switch on. Older iOS will mute it.
- **Measured levels** (desktop Chrome): the ambient music sits around −29 dBFS RMS, which is soft background level. The win flourish peaks at about 0.36. Even 19 chimes fired at once peaked at 0.94, with no clipping.

## 12. Saved data (localStorage)

Every read and write is wrapped in `try/catch`, and the game works fine without storage, for example in private browsing. Data stays on the player's device.

| Key | Value | Set by |
|---|---|---|
| `wordsearch-music` | `"1"` / `"0"` | Music button |
| `wordsearch-in-order` | `"1"` / `"0"` | In order switch |
| `wordsearch-crazy` | `"1"` / `"0"` | Crazy switch |
| `wordsearch-crazy-secs` | `"10"` … `"60"` | Seconds select |
| `wordsearch-list-filter` | a category name, or `"All"` | Puzzles list filter |
| `wordsearch-stats` | JSON, see §9 | `recordWin` |
| `umami.disabled` | `"1"` when set | `?analytics=off` (owner opt-out, §17). This is Umami's own key. |

If you change the stats schema, migrate the old data or start a new key. Don't silently break players' records.

## 13. Common changes

**Add or edit a puzzle**
1. Add the word list to `THEMES` (A–Z, 2–10 letters, at least about 30 words), and add the puzzle's name to one category in `CATEGORIES`.
2. Add a paragraph for **every** word to `words-info.js` under the same puzzle name.
3. **Bump the version** in `<script src="words-info.js?v=N">` in `index.html`. Otherwise returning players' browsers may keep the old cached file and show "No description yet" for the new words.
4. Run the coverage check and the fit check (§14). Aim for 20 words placed in about 100% of boards.

**Change the grid or word count:** `SIZE` and `WORDS_PER_GAME`. A 10×10 grid fits 20 words reliably; more needs a bigger grid or shorter words. The cell font size (`clamp(14px, 5.2vw, 26px)`) is tuned for 10 columns.

**Change highlight colours:** edit `COLORS`. Its order *is* the switching order (Rule 1). Check the overlap mixing on the light board.

**Change the music:** `CHORDS`, `PENTA`, `VOLUME`, and the filter frequencies in `Sound`.

## 14. Testing and verifying

There's no automated test suite. Changes are verified in a real browser at phone size (375 × 812) by driving the game from the browser console. These snippets assume the page is loaded at `http://localhost:8000/` (see "Run it locally" in `README.md`). They also work on the live site.

**Solve the current board.** This finds each word in the grid and fires a real pointer swipe:

```js
const N = 10, D = [[0,1],[1,0],[1,1],[-1,1],[0,-1],[-1,0],[-1,-1],[1,-1]];
const words = () => [...document.querySelectorAll('#words li')].map(l => l.dataset.word);
const grid = () => { const c = [...document.querySelectorAll('.cell')], g = [];
  for (let r = 0; r < N; r++) g.push(c.slice(r*N, r*N+N).map(x => x.textContent)); return g; };
const locate = (g, w) => { for (let r = 0; r < N; r++) for (let c = 0; c < N; c++) for (const [dr, dc] of D) {
  let ok = true; for (let i = 0; i < w.length; i++) { const y = r+dr*i, x = c+dc*i;
    if (y < 0 || y >= N || x < 0 || x >= N || g[y][x] !== w[i]) { ok = false; break; } }
  if (ok) return { r, c, r2: r+dr*(w.length-1), c2: c+dc*(w.length-1) }; } };
const swipe = h => { const rect = document.getElementById('grid').getBoundingClientRect(), cs = rect.width / N,
  b = document.getElementById('board'), pt = (r, c) => ({ clientX: rect.left+(c+.5)*cs, clientY: rect.top+(r+.5)*cs, pointerId: 1, bubbles: true });
  b.dispatchEvent(new PointerEvent('pointerdown', pt(h.r, h.c)));
  b.dispatchEvent(new PointerEvent('pointermove', pt(h.r2, h.c2)));
  b.dispatchEvent(new PointerEvent('pointerup', pt(h.r2, h.c2))); };
const g = grid(); for (const w of words()) swipe(locate(g, w));
```

**Checks worth running after changes:**
- **Fit:** play 30 or more boards per puzzle (via `#openList` → `[data-theme="…"]`) and count `#words li`. Expect 20.
- **Colour rules:** after solving, rebuild each highlight's segment from the found words and check Rules 2–4 against an *independent* overlap test. For example, sample the distance between segments and treat anything under the 0.7 bar width as touching.
- **Coverage:** every word in `THEMES` has an entry in `WORD_INFO` under the same puzzle, and there are no leftover entries.
- **Layout:** at 375 × 812, `document.documentElement.scrollWidth` should be 375 (no sideways scroll), and the page should fit without vertical scrolling while playing.
- **Timing:** use real waits (`await new Promise(r => setTimeout(r, ms))`) to check that the crazy clock and stopwatch pause while a dialog is open.
- **Audio:** you can't hear the browser pane, so measure instead. Patch `AudioNode.prototype.connect` *before* turning music on so the output is also routed into an `AnalyserNode`, then read RMS and peak levels. The music button needs a real click, not `element.click()`, for the browser to allow audio.
- **Cache:** the local server caches pages, so add a query string (`?v=2`) when reloading after edits.

## 15. Known limitations

- **Records stay on the device:** they don't sync between devices, and clearing site data or using a private window resets them. Syncing would need a backend.
- **"The Office: Surnames"** averages about 18 words per board rather than 20, because the names are long.
- **No keyboard or screen-reader way to play the grid:** finding words needs a pointer.
- **No dark mode:** the multiply-blended highlights rely on a light board (see `design.md`).
- **The viewport disables pinch zoom** (`user-scalable=no`) so swipes aren't mistaken for zooms. That's an accessibility trade-off.
- **Chance copies of short words:** these can appear in the filler letters, and either copy counts.

## 16. Decision log

These are product decisions, made deliberately with the product owner. Keep them unless the owner changes them.

| Decision | Why |
|---|---|
| 10×10 grid with 20 words | Matches the owner's puzzle book; packed boards are "more interesting and colourful". |
| Maximum-overlap placement | Without it, only about 10% of words crossed and most boards stayed one colour; with it, most boards use 4–6 colours. |
| Words inside words allowed | The owner considers hidden nested words part of the fun. |
| Colour Rules 1–4 as in §6 | Owner-specified. Rule 2 is per word, not per board; Rule 4 is kept even though it partly overlaps Rule 2. |
| Overlap = bars touching (geometric), not shared letters | Diagonal X crossings with no shared letter must count; the owner reported a KOALA highlight that should have switched colour. |
| Highlight opacity 0.5 with multiply | Overlaps must stay visible and mix into new colours. |
| Word list alphabetical; Puzzles list alphabetical | Owner request. |
| Crazy reset = new board, same puzzle | Keeping the same grid would let players memorise positions. |
| Crazy clock starts on first touch | Gives the player a moment to look at a fresh board. |
| Bests kept per mode combination; a mode must stay on all game | Fair comparisons; you can't earn a harder-mode record by switching it on late. |
| Generative music, off by default | The owner wanted music like 1in8b.life, which is code-generated; browsers block audio before a tap. |
| HAY replaced by MUG in "The Office: Moments" | No accurate description could be written for HAY. |

## 17. Analytics

The game sends anonymous, cookie-free gameplay events to Umami Cloud. **The full reference is `docs/analytics.md`**: events, fields, call points and testing. In short:

- **Script:** the Umami script in `<head>` only reports from `congph.com` (`data-domains`), so localhost never sends data.
- **Sending:** all events go through `track(name, data)`. It's guarded, so a blocked or failed script never affects play. `?debug-analytics` logs each call to the console.
- **Ten events,** about 2–5 per board: `game_started` (first swipe), then exactly one ending (`game_completed`, `game_abandoned` or `crazy_timeout`), plus toggles, word cards, and the Puzzles list and its filter.
  - `boardSource` / `boardStarted` track each board's lifecycle.
  - `newGame(theme, source)` takes the source (`load`, `new`, `list`, `replay`, `crazy_reset`; `resume` after a back/forward-cache restore).
- **Treat event names and fields like saved data:** renaming or removing them breaks the dashboards and history.
