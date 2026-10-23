# Design guide

This is the design reference for the Word Search game: what it looks like, why, and what any redesign has to preserve. It's written for designers and the AI agents they work with. All values are taken from the live CSS in `index.html`; sizes were measured at 375 × 812, the reference phone. For how the game logic works, see `how-it-works.md`.

## Contents

1. [Design principles](#1-design-principles)
2. [Hard constraints](#2-hard-constraints)
3. [Design tokens](#3-design-tokens)
4. [Layout](#4-layout)
5. [Components and states](#5-components-and-states)
6. [The highlight system](#6-the-highlight-system)
7. [Motion and haptics](#7-motion-and-haptics)
8. [Sound](#8-sound)
9. [Copy and voice](#9-copy-and-voice)
10. [Accessibility: current state and gaps](#10-accessibility-current-state-and-gaps)
11. [Open design opportunities](#11-open-design-opportunities)
12. [Working on the design](#12-working-on-the-design)

---

## 1. Design principles

1. **The board is the hero.** Everything else is quiet: a warm paper background, white surfaces, one dark ink colour. The only loud colour on screen is the neon highlights.
2. **Like a printed puzzle book, but alive.** Packed grids, words crossing everywhere, highlighter-pen colours that mix where they overlap. A finished board should look beautiful.
3. **One-handed and phone-first.** Every interaction works with a thumb: swipe on the grid, tap chips, flip switches. The whole game fits on one phone screen without scrolling.
4. **Calm by default, intense when chosen.** Normal play is relaxed, with soft ambient music. "Crazy" mode adds urgency: a red countdown, ticking, a shake. That urgency is opt-in.
5. **Feedback for every action.** Each player action gets an immediate visual response, plus a sound and a vibration where the device supports them.

## 2. Hard constraints

A redesign must keep these. Each one is tied to game logic or a product decision.

| Constraint | Why |
|---|---|
| **Square 10 × 10 grid, cells of equal size.** | Selection maps pointer position straight to a cell, and highlights are drawn in cell units. |
| **The board must be a light, near-white surface.** | Highlights use `mix-blend-mode: multiply` to mix colours where they overlap. Multiply on a dark surface turns them muddy or invisible. A dark mode needs a different blending approach; see §11. |
| **Highlight bars stay under 0.707 cells thick** (currently 0.7). | The game decides which highlights "overlap", which drives the colour rules, by geometry that assumes this width. Thicker bars would visibly touch without the rules noticing. |
| **Highlight colour order is fixed:** yellow → green → orange → blue → purple → pink. | The order *is* a game rule: colours switch to the next one in this sequence. You can change the exact hex values, but keep six distinct, bright, multiply-friendly colours. |
| **Highlights sit behind the letters.** | Letters must stay readable through any number of layered highlights. |
| **No page scroll while swiping on the board** (`touch-action: none`). | Otherwise a swipe scrolls the page instead of selecting. |
| **Everything fits in 375 × 812 without scrolling during play.** | You need to see the grid and the word list together, especially in crazy mode. The vertical budget is in §4. |
| **Word list stays alphabetical,** and "In order" marks the next word. | The In order mode depends on it. |
| **Colour is never the only signal.** | Found words are also struck through; the urgent state also shows numbers. Keep it that way (§10). |

## 3. Design tokens

### Colour

CSS custom properties on `:root`:

| Token | Value | Used for |
|---|---|---|
| `--bg` | `#f4f1ea` | Page background (warm paper); dialog surfaces |
| `--panel` | `#ffffff` | Board, word chips, game modes card, puzzle cards, secondary buttons |
| `--ink` | `#1f2a44` | All primary text, grid letters |
| `--accent` | `#1f2a44` | Primary buttons, "on" switches, crazy timer fill, current-puzzle outline (same as ink) |
| `--muted` | `#8a8f9c` | Secondary text: theme line, status, switch labels, list word previews |
| `--line` | `#e3dfd5` | Borders, "off" switch track, empty timer track, neutral card band |
| `--select` | `rgba(80, 140, 255, 0.3)` | In-progress swipe bar |

Hard-coded colours, not yet tokens:

| Value | Used for |
|---|---|
| `#ff3b30` | Urgent (crazy clock ≤ 5 s): timer fill, clock text, time-up ring |
| `#1f8a43` | "✓ Completed" in the Puzzles list |
| `rgba(31, 42, 68, 0.7)` | Text on found chips (struck through) |
| `rgba(31, 42, 68, 0.45)` | Dialog backdrop |
| `#1f2a44` | `theme-color` meta (browser toolbar colour) |

**Highlight colours** (the `COLORS` list in the script, not CSS):

| Order | Name | Hex |
|---|---|---|
| 1 | Neon yellow | `#f2ff00` |
| 2 | Neon green | `#39ff14` |
| 3 | Neon orange | `#ff9500` |
| 4 | Neon blue | `#00d5ff` |
| 5 | Neon purple | `#c04dff` |
| 6 | Neon pink | `#ff2ec8` |

### Typography

The game uses the device's own font (`-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif`), so nothing has to load. Weights in use: 500, 600 and 700.

| Element | Size | Weight | Other |
|---|---|---|---|
| Title "Word Search" | 1.4rem (1.2rem under 400 px wide) | 700 | letter-spacing 0.02em, no wrap |
| Theme line | 0.9rem | 400 | muted |
| Grid letters | `clamp(14px, 5.2vw, 26px)`, 19.5 px on the reference phone | 700 | ink |
| Word chips | 0.9rem | 600 | letter-spacing 0.04em, upper case |
| Status line | 1rem | 400 | muted, centred |
| Status (win) | 1.1rem | 700 | ink |
| "GAME MODES" label | 0.72rem | 700 | upper case, letter-spacing 0.08em, muted |
| Switch labels | 0.9rem | 600 (`strong`) | ink |
| Buttons | 1rem | 600 | |
| Dialog title | 1.15rem | 700 | |
| Puzzle card title | 1rem | 700 | word count in 0.85rem muted |
| Puzzle card word preview | 0.85rem | 400 | muted, line-height 1.5 |
| Records line | 0.85rem | 400 / 700 for times | times use tabular figures |
| Word card: puzzle name | 0.8rem | 400 | upper case, letter-spacing 0.04em, muted |
| Word card: word | 1.5rem | 700 | letter-spacing 0.06em |
| Word card: paragraph | 1rem | 400 | line-height 1.55 |
| Clock "⏱ 12s" | inherits status | 400 (700 when urgent) | tabular figures |

### Shape, space and depth

| | Value |
|---|---|
| Pill radius (buttons, chips, switches) | 999px |
| Board radius | 16px |
| Card radius (game modes, puzzle cards) | 14px |
| Dialog radius | 18px |
| Select radius | 8px |
| Page side padding | 16px; max content width 560px, centred |
| Vertical gap between sections | 14px |
| Header gaps | 12px (8px under 400 px); 8px between buttons (6px) |
| Board inner padding | 8px |
| Chip padding / gap | 6px 12px / 8px |
| Board shadow | `0 2px 12px rgba(31, 42, 68, 0.08)` |
| Dialog shadow | `0 10px 40px rgba(0, 0, 0, 0.25)` |

## 4. Layout

One centred column, top to bottom:

```
┌──────────────────────────────────────────┐
│ Word Search            [♪] [Puzzles][New]│  header
│ Theme: Ocean                             │
├──────────────────────────────────────────┤
│ ┌──────────────────────────────────────┐ │
│ │  S D Q D N W A R P Y                 │ │  board (square)
│ │  …10 × 10 letters…                   │ │
│ │  ▬▬▬▬▬▬▬▬▬▬▬▬▬ (crazy timer bar)     │ │  ← only in crazy mode
│ └──────────────────────────────────────┘ │
│   3 of 20 found · ⏱ 12s                  │  status line + clock
│ ┌────────────── GAME MODES ────────────┐ │
│ │   (○) In order     (○) Crazy [30s▾]  │ │  game modes card
│ └──────────────────────────────────────┘ │
│ (ANCHOR) (BAY) (CLAM) (COD) (CORAL) …    │  word chips, wrapping
└──────────────────────────────────────────┘
```

**Vertical budget at 375 × 812** (measured; this is the tightest constraint):

| Block | Height | Top |
|---|---|---|
| Header | 42 | 16 |
| Board | 343 | 72 |
| Status (1 line; 2 lines ≈ 46) | 23 | 429 |
| Game modes card | 66 | 466 |
| Word chips (20 chips, about 6 rows) | 226 | 546 |
| Bottom padding | 16 | → total about 812 |

There's **no spare room**. Anything new on the main screen has to replace something, fold into an existing block, or go in a dialog. The crazy timer bar was placed inside the board's bottom padding for this reason.

**Wider screens:** the column caps at 560 px and centres. There's no tablet or desktop layout yet.

## 5. Components and states

### Header
- **Title and theme line:** the title is on the left, with "Theme: \<puzzle\>" in muted text below it.
- **Music button:** a 42 × 42 round, secondary button with a music-note icon.
  - **Off:** muted colour with a diagonal slash.
  - **On:** ink colour, no slash.
  - Uses `aria-pressed`.
- **Puzzles:** a secondary pill button (white with a border) that opens the Puzzles list.
- **New:** a primary pill button (filled ink) that starts a new random puzzle.
- **Pressed state:** all buttons scale to 0.97.

### Board
- **Structure:** a white rounded card with a soft shadow. Inside is a square area with the letter grid on top and the highlight overlay beneath it.
- **States:**
  - **Idle:** letters only.
  - **Swiping:** a light-blue bar from the start letter to the snapped end letter. It snaps to the 8 straight directions.
  - **Found:** neon highlight bars (§6).
  - **Crazy mode:** a 4 px timer bar along the bottom inside padding. Dark fill on a `--line` track, shrinking left to right.
  - **Urgent** (≤ 5 s): the timer fill turns `#ff3b30`.
  - **Time up:** a 0.6 s shake with a 4 px red ring, then a new board.
  - **Solved:** further swipes are ignored.

### Status line
- **What it shows:** a single centred line of muted text giving instructions, progress, errors and the result. The crazy clock "⏱ Ns" is appended after " · " and turns red and bold when urgent.
- **Win state:** larger, bold ink text (`.status.win`).
- **Messages:** see §9.

### Game modes card
- **Structure:** a white rounded card with the small "GAME MODES" label, holding two switches side by side.
- **Switch:** a custom 40 × 24 track.
  - **Off:** `--line` track, white knob on the left.
  - **On:** `--accent` track, knob slides right.
  - Animated over 0.2 s.
- **Crazy seconds select:** a small native select (10–60 s), placed next to the Crazy switch.

### Word chips
- **Default:** a white pill with a hairline border and upper-case text. The whole chip can be tapped and opens the word card.
- **Pressed:** scales to 0.96.
- **Found:** the background becomes the word's highlight colour, the border disappears, and the text is struck through at 70% ink. The background fades over 0.2 s.
- **Next (In order mode):** a 2 px ink outline (border plus 1 px ring) on the next word to find.
- **Rejected (out-of-order swipe):** a 0.35 s horizontal shake.

### Analytics footnote
- **Look:** one centred line of small muted text (0.72rem) under the word chips: "This game uses anonymous, cookie-free analytics (Umami). No personal data is collected." The link to umami.is uses the same colour as the text.
- **Position:** it sits after the chips, so on short screens it's just below the fold when the word list wraps to 7 rows. That's fine for a footnote: it never pushes the board or chips.

### Puzzles list (dialog)
- **Frame:** a modal sheet on the paper background, at most 560 px wide.
- **Sticky header:** the "Puzzles" title, a Close button, and the filter row underneath.
- **Filter row:** one row of pill chips that scrolls sideways (no scrollbar): "All", then each category, each with a faded count ("Food & Drink 6").
  - **Selected chip:** filled ink with white text.
  - **Other chips:** white with a hairline border.
  - **Memory:** the chosen filter is remembered.
- **Category sections:** a small upper-case muted heading (0.78rem, letter-spacing 0.08em, like "GAME MODES"), with "N of M completed" on the right, then that category's cards.
- **Puzzle card:** white and rounded, showing:
  - the name;
  - a Play button, which reads "Replay" for the current puzzle;
  - a folded "▸ N words" toggle that reveals the full word list as muted text.
- **Current puzzle:** a 2 px ink border, and the list opens scrolled to it.
- **Completed puzzle:** a records line, "✓ Completed ×N" in green plus "Best: 1:58 Normal · 2:40 Crazy". There's one time per mode combination played.
- **Order and closing:** categories follow their set order; cards within a category are alphabetical. Tapping the backdrop closes the dialog.

### Word card (dialog)
- **Frame:** a compact modal, at most 420 px wide.
- **Contents:**
  - a 10 px colour band at the top, in the word's highlight colour if found, or neutral `--line` if not;
  - the puzzle name in small caps;
  - the word, large;
  - the paragraph;
  - a full-width primary Close button.
- **Closing:** tapping the backdrop closes it.

## 6. The highlight system

This is the signature of the game, so it gets its own section.

- **Shape:** a straight bar from the centre of the first letter to the centre of the last, 0.7 cells thick, with fully round ends. It looks like a highlighter stroke.
- **Look:** neon colour at 50% opacity with `mix-blend-mode: multiply`. Where highlights cross, the colours combine the way ink does: yellow and blue make green, yellow and pink make red-orange, and same-colour crossings get darker.
- **Readability:** letters are always drawn above the highlights. Ink text over a 50% highlight on white has a contrast of at least 7.3:1 for every colour.
- **Which colour:** the game decides automatically, by rule. In short, it stays on one colour and moves to the next only when needed to keep crossing words distinguishable. The full rules are in `how-it-works.md` §6. As a designer you can change *how* highlights look, but they must keep:
  - six distinct colours in the fixed order;
  - visible overlaps;
  - readable letters.
- **Selection bar:** the same shape in a translucent blue, so it reads as "in progress" rather than "found".

## 7. Motion and haptics

| Moment | Motion | Haptic (Android; iOS ignores vibration) |
|---|---|---|
| Button press | scale 0.97 | – |
| Chip press | scale 0.96 | – |
| Word found | chip background fades in, 0.2 s | 30 ms |
| Out-of-order swipe | chip shakes ±4 px, 0.35 s | 20-40-20 ms |
| Crazy time up | board shakes (−6, +6, −4, +3 px) with a red ring, 0.6 s; new board 1.1 s later | 80-60-80 ms |
| Switch toggle | track colour and knob slide, 0.2 s | – |
| Crazy timer | bar shrinks continuously, updated every frame | – |
| Music on / off | fades in over 1.5 s, out over 0.6 s | – |

There's no `prefers-reduced-motion` handling yet (§10).

## 8. Sound

Sound is part of the experience, not decoration. Everything is synthesised live, with no audio files. The music is **off by default**, and the header button toggles it; the choice is remembered.

- **Ambient music:** warm, slow and drifting, like a calm puzzle book on a rainy afternoon.
  - **Chords:** soft rolled chords change every 8 seconds, in a loop of about a minute: Fmaj7 · Em7 · Dm7 · Cmaj7 · Am7 · Dm9 · Gsus · G.
  - **Melody:** random bell-like notes from a pentatonic scale drift over the chords, so it never repeats exactly.
  - **Tone:** a muffled filter and a long reverb.
- **Reactive layer:**

  | Moment | Sound |
  |---|---|
  | Progress | The melody gets busier and higher as more of the board is found. |
  | Word found | A quick three-note rising chime, a step higher with each word. |
  | Out-of-order | A soft, low two-note "nope". |
  | Crazy urgent | A gentle tick each second; the music gets brighter. |
  | Time up | Four falling notes. |
  | Board solved | A sweep up the scale into a big soft chord. |

Any new sound should match this palette: soft sine and triangle tones, gentle attacks, long fades, no harsh or percussive sounds.

## 9. Copy and voice

The voice is short, friendly and plain. There's one emoji per moment at most, saved for big moments (🎉 ⏰). Words in the game are shown in upper case.

| Where | Text |
|---|---|
| Start of a board | Swipe across letters to find words. Tap a word below to learn about it. |
| Progress | 3 of 20 found |
| Out of order | Not yet — find KELLY first |
| Crazy switched on | Crazy mode — the clock starts on your next swipe. |
| Crazy time up | ⏰ Time's up! Starting over… |
| After a crazy reset | New board — tap to start the clock. |
| First finish | 🎉 Done in 2:14 — first finish! |
| New best | 🎉 Done in 1:58 — new personal best! |
| Not a best | 🎉 Done in 2:30 · best 1:58 |
| Mode suffix | (In order), (Crazy), (In order + Crazy) |
| Puzzles list record | ✓ Completed ×3 · Best: 1:58 Normal |
| Word card with no text | No description yet for this word. |
| Category progress | 2 of 6 completed |
| Analytics footnote | This game uses anonymous, cookie-free analytics (Umami). No personal data is collected. |

**Word card paragraphs** are one or two plain, factual sentences, about 15–40 words, in British-leaning spelling ("colour", "favourite") to match the rest of the copy.

## 10. Accessibility: current state and gaps

**What already works:**
- Primary text contrast is high: ink on paper 12.6:1, ink on white 14.3:1.
- Grid letters over any highlight have a contrast of at least 7.3:1.
- Found words are struck through, not only coloured, and the urgent clock shows numbers, not only red.
- The music button exposes its state (`aria-pressed`, `aria-label`); the seconds select is labelled; the dialogs use native `<dialog>` (focus handling, Esc to close).
- Every interaction works with a single finger, with no multi-touch.

**Known gaps** (good targets for design work):

| Gap | Detail |
|---|---|
| Muted text contrast | `--muted` on paper is 2.9:1 and on white 3.2:1, below the 4.5:1 guideline for small text. It's used for the status line, theme line, switch labels and list previews. |
| Found-chip text on some colours | Struck-through text on the chip colour: purple 2.7:1, pink 3.1:1, orange 3.8:1 (yellow 5.3, green 4.8, blue 4.1). |
| Urgent red | `#ff3b30` on paper is 3.1:1. |
| Touch targets | Word chips are 31 px tall and switches 24 px tall, below the roughly 44 px guideline. |
| No keyboard or screen-reader play | The grid can only be played by swiping. |
| Zoom disabled | The viewport sets `user-scalable=no` so swipes aren't read as zooms. |
| No reduced-motion mode | The shakes and slides ignore `prefers-reduced-motion`. |
| No dark mode | Blocked by the multiply highlights (§2). |

## 11. Open design opportunities

These are ideas, not commitments; check with the product owner first.

- **Dark mode.** You'd keep a light board "page" on a dark surround, or redesign the highlights for dark (for example `screen` blending with adjusted neon colours). Both would need testing against the colour rules.
- **A win moment.** Right now the win is only the status text plus the music flourish. A small celebration, or a record card showing time vs best, could make it feel bigger.
- **Puzzle list as a visual library.** For example, cards with a tiny solved-board thumbnail or a theme illustration.
- **Accessible contrast pass.** Darken `--muted`, and choose found-chip text colours per highlight colour.
- **Larger touch targets** for chips and switches, without breaking the no-scroll budget (§4).
- **Tablet and desktop layout** using the extra width (for example, the word list beside the board).
- **Theme branding,** for example a distinct accent per puzzle such as The Office's, within the constraints.

## 12. Working on the design

- **To see it:** run it locally (see `README.md`) and open it at 375 × 812, using browser devtools' device mode. Turn on music, both modes, and play a full board to see every state.
- **To change it:**
  - Colours and spacing live in the `<style>` block of `index.html`.
  - Highlight colours are `COLORS` in the script.
  - Copy strings are in the script, in the places listed in `how-it-works.md`.
- **Before handing back a change**, check:
  1. The page still fits 375 × 812 with no scroll while playing (and no sideways scroll).
  2. Crossing highlights still visibly mix, and the letters stay readable.
  3. Every state in §5 still has a clear look: next, nope, found, urgent, time-up, win, completed.
  4. The constraints in §2 still hold.
