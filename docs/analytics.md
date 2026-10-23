# Analytics

**Status:** implemented with Umami Cloud. This is the reference for what the game tracks and why. The code is in `index.html`; search for `track(`.

## 1. Goals

Answer three questions:

1. **Where are players from?** Country (reliable) and region or city (approximate), plus language, device, and where they came from (the portfolio, a shared link, search).
2. **How do they play?** How often, how long, which modes, whether they finish or give up, and whether they use the music and word cards.
3. **Which puzzles do they play?** Starts, completions, completion rate and typical finish time, per puzzle.

## 2. Principles

- **Anonymous:** no names, emails, accounts, IP addresses or anything else personal. The only identifiers are the ones the analytics service creates itself, and both candidates below avoid cookies.
- **No cookie banner:** the game has no screen space for one, and a cookie-free service shouldn't need it in most places. Confirm this for your own situation.
- **Track games, not keystrokes:** about 2–5 events per board. There's no event per word found or per swipe.
- **Tracking never affects play:** if the script is blocked (ad blockers) or fails, the game behaves exactly as it does today.
- **Easy to switch services:** all tracking goes through one small `track()` helper, so changing service means changing one function.

## 3. Choosing a service (decision needed)

| | Umami Cloud | Plausible |
|---|---|---|
| Cost | Free plan for small sites | Paid subscription |
| Cookies | None | None |
| Location | Country, region, city | Country, region, city |
| Custom events with details | Yes | Yes |
| Dashboard | Good | More polished, with easy filtering by event details |
| Script size | About 2 KB | About 1 KB |

**Chosen: Umami Cloud.** The site ID is `901a1081-aadb-4708-8d88-bf34261b4ac0`; it's public by design, since it's in the page. Moving to Plausible later only means changing the `track()` helper, though past data wouldn't transfer.

The script tag has `data-domains="congph.com"`, so **local copies (localhost) never send data**.

## 4. Automatic data (no code beyond the script tag)

Recorded for every page view: **country, region, city** (worked out from the connection; city is approximate), **language, device type, browser, operating system, screen size, referrer** (the site the visitor came from), and the **date and time**. The services also count **unique and returning visitors** without cookies, using an anonymised daily identifier.

This answers question 1 by itself.

## 5. Custom events

Values used in event details:

| Field | Values |
|---|---|
| `puzzle` | Puzzle name exactly as in `THEMES`, for example `Ocean`, `The Office: Characters` |
| `category` | The puzzle's category, as in `CATEGORIES`, for example `Food & Drink` |
| `modes` | `normal`, `order`, `crazy`, `order+crazy`: the same keys as the personal-best records (`modeKey`) |
| `source` | How the board appeared: `load` (opening the page), `new` (New button), `list` (picking from the Puzzles list), `replay` (Replay in the list), `crazy_reset` (crazy timer ran out), `resume` (the page was closed mid-game, then restored by the browser's back/forward cache) |
| `seconds` | Whole seconds, rounded |
| `found` / `total` | Number of words found / on the board |

### The events

| Event | When it fires | Details | Answers |
|---|---|---|---|
| `game_started` | First swipe on a new board. Boards nobody touched don't count. | `puzzle`, `category`, `modes`, `source` | Q3: which puzzles get played; Q2: how people start games |
| `game_completed` | Last word found | `puzzle`, `category`, `modes`, `seconds`, `first_finish` (yes/no), `new_best` (yes/no) | Q3: completions and finish times; Q2: improvement over time |
| `game_abandoned` | A started, unfinished board is replaced (New, picking a puzzle, Replay), **or the page is closed or left mid-game** | `puzzle`, `modes`, `found`, `total`, `seconds`, `closed` (yes = page closed or left; no = replaced in-game) | Q2: where and when people give up |
| `crazy_timeout` | Crazy timer runs out | `puzzle`, `found`, `total`, `crazy_seconds` (10–60) | Q2: whether the timer is too harsh |
| `mode_toggled` | In order or Crazy switched | `mode` (`order`/`crazy`), `on` (yes/no) | Q2: how popular each mode is |
| `crazy_seconds_changed` | Crazy seconds setting changed | `crazy_seconds` | Q2: which difficulty players prefer |
| `music_toggled` | Music button tapped | `on` (yes/no) | Q2: whether people want music |
| `word_card_opened` | A word chip tapped | `puzzle`, `word`, `found` (yes/no) | Q2: whether the paragraphs get read, and which words interest people |
| `puzzles_list_opened` | Puzzles button tapped | – | Q2: how people choose puzzles |
| `puzzles_filter_changed` | A category chip tapped in the Puzzles list | `category` (or `All`) | Q3: which categories people browse |

**How they relate:**
- **Every `game_started` ends exactly one way:** `game_completed`, `game_abandoned`, or `crazy_timeout`. A crazy reset then starts a new board with `source: crazy_reset`.
- **Closing the page mid-game** sends `game_abandoned` with `closed: true`, from the browser's `pagehide` event. Browsers sometimes drop data sent while a page closes (especially phones killing a backgrounded tab), so a few of these will be missing. Treat starts with no ending as "left the site".
- **Completion rate per puzzle** = `game_completed` ÷ `game_started`, both filtered by `puzzle`.
- **Volume:** a typical visit sends 1 page view plus about 3–6 events, well within free-plan limits for a personal project.

### Deliberately not tracked

- **Individual words found or swipes:** too many events for too little insight.
- **Mistaken swipes and out-of-order rejections:** could be added later if difficulty tuning needs them.
- **Anything typed, any identifier from `localStorage`, or players' stored best times:** personal records stay on the device.

## 6. Questions and where the answers come from

| Question | How to read it |
|---|---|
| Where are my players from? | Dashboard → Locations (page views) |
| Where do they come from (portfolio, social, search)? | Dashboard → Referrers |
| Phones or computers? | Dashboard → Devices |
| How many play each day or week, and how many return? | Visitors over time, plus returning visitors |
| How many boards does a visit involve? | `game_started` count ÷ visits |
| Which puzzles are most popular? | `game_started` broken down by `puzzle` |
| Which puzzles get finished? | `game_completed` ÷ `game_started`, by `puzzle` |
| How long does a puzzle take? | `game_completed` → average `seconds`, by `puzzle` and `modes` |
| Where do people give up? | `game_abandoned` → `found` out of `total` |
| Is crazy mode too hard? | `crazy_timeout` vs crazy-mode `game_completed`; most common `crazy_seconds` |
| Do people use the modes, music and word cards? | `mode_toggled`, `music_toggled`, `word_card_opened` |
| Which words do people look up? | `word_card_opened` broken down by `word` |

## 7. How it's built

All in `index.html`, with no build step and no new files.

- **Script tag** in `<head>`, loaded `defer` (in the background) from `cloud.umami.is`, with `data-domains="congph.com"`.
- **`track(name, data)`** calls `umami.track` if the script has loaded, inside `try/catch`, so tracking never throws into the game and does nothing when blocked.
- **Board state:** `boardSource` records how the board appeared, and `boardStarted` makes `game_started` fire once per board, on the first swipe. `newGame(theme, source)` reports the previous board as `game_abandoned` if it was started and unfinished. `timeUp()` reports `crazy_timeout` and clears `boardStarted`, so the reset doesn't also count as abandoned. A `pagehide` listener reports closing mid-game.
- **Call points:**

  | Event | Call point |
  |---|---|
  | `game_started` | the `pointerdown` handler (alongside `watchStart`) |
  | `game_completed` | after `recordWin()` |
  | `game_abandoned` | `newGame()` and `pagehide` |
  | `crazy_timeout` | `timeUp()` |
  | `mode_toggled`, `crazy_seconds_changed` | the switch and select `change` handlers |
  | `music_toggled` | the music button's click handler only (not the automatic restart of a saved "on") |
  | `word_card_opened` | the word-chip click handler |
  | `puzzles_list_opened` | the Puzzles button handler |
  | `puzzles_filter_changed` | the filter row's click handler |

- **Debug mode:** add `?debug-analytics` to the URL to log every `track()` call to the console. It works on localhost, where nothing is sent.
- **Opting out (the owner's own devices):** open `congph.com/wordsearch/?analytics=off` once in each browser you use. That sets Umami's own `umami.disabled` flag in the browser's storage, and Umami then sends nothing from that browser. `?analytics=on` removes it. Clearing the browser's site data removes it too.
- **Privacy notice:** a footnote under the word list ("This game uses anonymous, cookie-free analytics (Umami). No personal data is collected."), plus the Privacy section in `README.md`.

## 8. Testing

Tested before release (localhost, with `?debug-analytics`):

- An untouched board sends nothing.
- These sequences each sent the right events in the right order, with the correct details:
  - start a board, then press New → `game_started` + `game_abandoned`;
  - pick a puzzle from the list → `source: list`;
  - Replay → `source: replay`, with `first_finish` / `new_best` correct;
  - crazy time-out → `crazy_timeout`, then the reset board starts with `source: crazy_reset`;
  - closing the page mid-game → `game_abandoned` with `closed: true`, sent only once; restoring the page then starts with `source: resume`;
  - toggles, the seconds setting and word cards.
- `?analytics=off` and `=on` set and clear `umami.disabled`.
- No data is sent from localhost (only the script file itself loads).

After deploying, check Umami's real-time view while playing one board on a device that hasn't opted out.

## 9. Decisions

1. **Closing the page mid-game counts as giving up:** yes, `closed: true` (see §5).
2. **Location detail:** country, region and city (Umami's defaults).
3. **Privacy note:** in the README and as a small footnote at the bottom of the game.
4. **Owner's own visits are excluded,** via `?analytics=off` on each of the owner's browsers (see §7).
