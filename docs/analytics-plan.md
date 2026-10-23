# Analytics plan

**Status:** proposal, nothing implemented yet. Once it's built, this file becomes the reference for what the game tracks and why.

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

Either one covers every event below. **Recommendation: start with Umami Cloud's free plan.** Moving to Plausible later only means changing the `track()` helper; past data wouldn't transfer. Plans and limits change, so check both services' current pricing before signing up.

The owner needs to: create the account, add the site `congph.com` with path `/wordsearch/`, and share the **site ID**. It's a public value that ends up in the page, not a secret.

## 4. Automatic data (no code beyond the script tag)

Recorded for every page view: **country, region, city** (worked out from the connection; city is approximate), **language, device type, browser, operating system, screen size, referrer** (the site the visitor came from), and the **date and time**. The services also count **unique and returning visitors** without cookies, using an anonymised daily identifier.

This answers question 1 by itself.

## 5. Custom events

Values used in event details:

| Field | Values |
|---|---|
| `puzzle` | Puzzle name exactly as in `THEMES`, for example `Ocean`, `The Office: Characters` |
| `modes` | `normal`, `order`, `crazy`, `order+crazy`: the same keys as the personal-best records (`modeKey`) |
| `source` | How the board appeared: `load` (opening the page), `new` (New button), `list` (picking from the Puzzles list), `replay` (Replay in the list), `crazy_reset` (crazy timer ran out) |
| `seconds` | Whole seconds, rounded |
| `found` / `total` | Number of words found / on the board |

### The events

| Event | When it fires | Details | Answers |
|---|---|---|---|
| `game_started` | First swipe on a new board. Boards nobody touched don't count. | `puzzle`, `modes`, `source` | Q3: which puzzles get played; Q2: how people start games |
| `game_completed` | Last word found | `puzzle`, `modes`, `seconds`, `first_finish` (yes/no), `new_best` (yes/no) | Q3: completions and finish times; Q2: improvement over time |
| `game_abandoned` | A started, unfinished board is replaced (New, picking a puzzle, Replay). See open question 9.1 for closing the page. | `puzzle`, `modes`, `found`, `total`, `seconds` | Q2: where and when people give up |
| `crazy_timeout` | Crazy timer runs out | `puzzle`, `found`, `total`, `crazy_seconds` (10–60) | Q2: whether the timer is too harsh |
| `mode_toggled` | In order or Crazy switched | `mode` (`order`/`crazy`), `on` (yes/no) | Q2: how popular each mode is |
| `crazy_seconds_changed` | Crazy seconds setting changed | `crazy_seconds` | Q2: which difficulty players prefer |
| `music_toggled` | Music button tapped | `on` (yes/no) | Q2: whether people want music |
| `word_card_opened` | A word chip tapped | `puzzle`, `word`, `found` (yes/no) | Q2: whether the paragraphs get read, and which words interest people |
| `puzzles_list_opened` | Puzzles button tapped | – | Q2: how people choose puzzles |

**How they relate:**
- **Every `game_started` ends exactly one way:** `game_completed`, `game_abandoned`, or `crazy_timeout`. A crazy reset then starts a new board with `source: crazy_reset`. If the page is closed mid-game, it may end with no event at all (open question 9.1).
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

## 7. Implementation outline

All in `index.html`; there's no build step or new files.

1. **Script tag** in `<head>`, loaded `defer` (in the background) from the chosen service, with the site ID.
2. **`track(name, details)` helper** inside the IIFE. It calls the service's event function if present and otherwise does nothing, wrapped in `try/catch` so tracking can never throw into the game.
3. **Track the game's lifecycle** with a small amount of new state:
   - `boardSource`, set wherever a board is created: page load, New button, Puzzles list (Play vs Replay), crazy reset;
   - `startedTracked`, so `game_started` fires once per board, on the first swipe.

   `game_abandoned` fires in `newGame()` when the previous board was started but not finished.
4. **Call points** in existing code:

   | Event | Call point |
   |---|---|
   | `game_started` | the `pointerdown` handler (alongside `watchStart`) |
   | `game_completed` | after `recordWin()`, which already gives `ms`, `key` and `prev` |
   | `crazy_timeout` | `timeUp()` |
   | `mode_toggled`, `crazy_seconds_changed` | the switch and select `change` handlers |
   | `music_toggled` | `setSound()` |
   | `word_card_opened` | the word-chip click handler |
   | `puzzles_list_opened` | the Puzzles button handler |

5. **Docs:** add an "Analytics" section to `docs/how-it-works.md`, and update `AGENTS.md` so the event names and fields are treated like saved data: don't rename or remove them, because that breaks dashboards and history.

## 8. Testing before release

- **Locally:** with the service's script absent or blocked, the game plays exactly as before, with no console errors.
- **Events sent:** a temporary debug mode logs each `track()` call to the console. Play one board of each kind (completed, abandoned, crazy timeout, Replay) and check the names, fields and the "one ending per start" rule.
- **Live:** after pushing, confirm the events appear in the service's real-time view. Both services can ignore your own visits, which is worth turning on so testing doesn't skew the numbers.
- **Performance:** the page still loads and fits as before. The script loads in the background and doesn't delay the board.

## 9. Open questions

1. **Closing the page mid-game:** report `game_abandoned` when the page is hidden or closed? Browsers don't always deliver data sent at that moment, so some would be lost. Option: accept that unfinished games without an ending event count as "left the site".
2. **Region and city:** keep them, or record country only (more private, and still answers question 1)?
3. **Privacy note:** add a short line in the game or README, such as "This game uses cookie-free, anonymous analytics (Umami)"? Recommended for transparency.
4. **Your own visits:** exclude them using the service's "ignore my visits" option, so your testing doesn't count.
