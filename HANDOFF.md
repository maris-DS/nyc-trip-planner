# Handoff — NYC + Camp trip planner

Everything a fresh session needs to keep working on this without undoing
decisions that were already made deliberately.

---

## 1. What this is

A single self-contained HTML page planning one trip: **New York + Camp No
Counselors, Sept 1–8 2026**, for two people (Maris and Scott). No build step, no
dependencies, no framework. Open the file and it works.

It started as a static itinerary and became a small scheduling app: booked
things are anchors, everything else reflows around them.

## 2. Where things are

| | |
|---|---|
| **Working file** | `~/Desktop/CLAUDE/PROJECTS/NYC Trip/site/index.html` |
| **Repo** | `~/Desktop/CLAUDE/PROJECTS/NYC Trip/site/` (git, remote `origin`) |
| **GitHub** | https://github.com/maris-DS/nyc-trip-planner (public) |
| **Live** | https://maris-ds.github.io/nyc-trip-planner/ |
| **Old versions** | `../nyc-trip.html`, `../nyc-camp-trip-planner.html` — superseded, ignore |
| **Original source** | `~/Downloads/nyc-camp-trip-planner (2).html` — the file Maris started with. **Use it to verify provenance** (see §6). |

**Deploy:** any push to `main` redeploys in ~30–60s.

```bash
cd "/Users/marisnaylor/Desktop/CLAUDE/PROJECTS/NYC Trip/site"
git add -A && git commit -m "..." && git push
```

Verify a deploy landed by diffing the live page against local:

```bash
curl -s https://maris-ds.github.io/nyc-trip-planner/ -o /tmp/live.html && diff -q /tmp/live.html index.html
```

**Always syntax-check before committing** — the file is one big inline script:

```bash
node --check <(sed -n '/^<script>/,/^<\/script>/p' index.html | sed '1d;$d')
```

## 3. Architecture

All in `index.html`. Two data structures near the top of `<script>`:

- **`DAYS[]`** — 8 days, each `{dow, num, kind, title, sub, stops:[...]}`.
- **`POOL[]`** — the candidate list, grouped in tiers. Everything offered as an
  option comes from here.

### Stop fields

| Field | Meaning |
|---|---|
| `t` | Time string, e.g. `"9:00–10:00 AM"`. **See §5 gotcha 1.** |
| `n` | Name. Prefixes like `SUBWAY —` / `DINNER —` are stripped by `cleanTitle()`. |
| `d` | Description shown when the row is expanded. |
| `la` / `lo` | Coordinates. |
| `k` | `air` \| `transit` \| `ticket` \| `hotel` \| `food` \| `event`. |
| `q` | **Google Maps search query — name first, then address.** Drives every link. |
| `c` | Cost per person. |
| `dur` | Explicit duration in minutes, overrides the parsed `t` range. |
| `fix` | Force anchor (used on the camp coaches + the NJT train). |
| `flex` | Force NOT an anchor (used on getting-ready blocks + "back at hotel"). |
| `pick` | Undecided slot — renders as "pick a spot" with options. |
| `paid` | Prepaid label; **excluded from the budget**. |
| `routes[]` | Multi-option transit block (see §4). |
| `steps[]` | Sub-steps shown when a transit row is expanded. |

### Key functions

- **`schedule(d)`** → `{items, gaps}`. The scheduling engine. Walks the day,
  assigns start/end times, flags overruns (`it.late`).
- **`isAnchor(s)`** — `flex` → false, `fix` → true, else `air|ticket|hotel`.
- **`stopsOf(i)`** — base stops + parked extras + resolved `pick` slots, in
  user order.
- **`computePins()`** — **the single source of truth for pin numbers.** Both
  the card badges and the map read it. Never compute numbers anywhere else.
- **`mapStops()`** — what the map shows. Excludes `air` and `transit` by design.
- **`suggestFor(d, point, kindFirst, meal)`** — option candidates from `POOL`.
- **`renderDays()` / `paintMap()`** — the two renderers.

### Storage keys (all `localStorage`)

Suffixes are **not** uniform — check the actual string before assuming.

| Store | Key | Keyed by |
|---|---|---|
| Saved-places checkboxes | `nyc-camp-picks-v1` | place slug |
| Checklist ticks | `nyc-camp-check-v1` | check `id` |
| Custom checks | `nyc-camp-check-custom-v1` | — |
| Parked extras | `nyc-camp-extras-v1` | day index → item objects |
| UI state | `nyc-camp-ui-v2` | — |
| Map on/off | `nyc-camp-nomap-v1` | — |
| **Costs** | `nyc-camp-costs-v1` | **`day:stopIndex`** |
| **Notes** | `nyc-camp-notes-v1` | **`day:stopIndex`** |
| **Durations** | `nyc-camp-durs-v3` | **`dur:day:bi`** |
| **Pins** | `nyc-camp-pins-v3` | **`day/b<bi>`** |
| **Order** | `nyc-camp-order-v3` | **day → `[itemId]`** |
| **Routes** | `nyc-camp-routes-v3` | **`day:blockIndex`** |
| **Slot picks** | `nyc-camp-slotpicks-v3` | **`day:blockIndex`** |

**Seven stores are position-keyed, not seven minus two.** The bolded rows all key
off a stop's index in `DAYS[d].stops`, so restructuring a day silently reattaches
old values to different rows. This bit us once: St. Marks inherited a 60-minute
duration meant for a different row.

Two ways to handle it, and the right one depends on what the store holds:

- **`durs` / `pins` / `order` / `routes` / `slotpicks`** — bump the `-vN` suffix.
  These hold tweaks that are cheap to redo, and a bump is one edit. It resets
  every day, not just the one you changed; that's the accepted trade. Bumped to
  `-v3` on 2026-07-27 for the Wednesday reshape. Old `-v1`/`-v2` entries are left
  orphaned in `localStorage` — harmless, never read.
- **`costs` / `notes`** — do **not** bump. These hold typed input worth keeping,
  and a global bump would wipe every day to fix one. Add a flagged one-time
  migration that deletes only the affected `day:index` keys instead. See the
  `nyc-camp-mig-wed-westside` IIFE near the top of the script for the pattern —
  copy it, change the flag name and the index range.

## 4. Features, and why they work the way they do

- **Anchors vs float.** Booked things (flights, ticketed entries, hotel
  check-in/out, camp coaches) have a fixed start. Everything else follows the
  cursor. Anchors also have an adjustable *duration* — the start is fixed, the
  length is just a number that might be wrong.
- **The "not before" floor applies to `food` only.** Meals hold their planned
  time so dinner doesn't drift to 3:20pm after an early check-in. Everything
  else must follow the cursor in **both** directions, or shortening a block
  silently changes nothing and the duration control feels broken.
- **Pick slots.** Meals with no reservation. Options come from `POOL`, filtered
  by meal type, ranked by walking distance, shown as full-width cards with a
  description and a **Reviews** link. Picking one renames the row, moves its
  map pin, and updates the directions link on the transit legs either side.
- **Meal types:** `breakfast` / `lunch` / `dinner` / `drinks` / `snack`. The
  bridge is one-way — a dinner restaurant is fine at lunch, a lunch counter is
  not an answer to "where should we have dinner". `snack` draws from breakfast
  and lunch.
- **Transit route pickers.** Four blocks (JFK→city, EWR→Hilton, Hilton→JFK,
  Romer→Newark) offer real alternatives with time and fare. Picking one changes
  that leg's duration and cost. **Live data is impossible** — a static file
  can't call Google's API — so each option deep-links to Google Maps instead.
- **Map.** Pan, pinch/wheel zoom, geolocation, per-pin walking directions.
  Numbers only, no text labels. Flights and transit are deliberately excluded.
  Markers group **strictly by coordinate** — three things at the same hotel are
  one pin labelled `1/7/9`. Never key markers by anything else or they stack.
- **Edit mode.** Drag-to-reorder is **off by default** behind an "Edit order"
  toggle. Scott reordered his day by accident trying to scroll.
- **Budget.** Per-person, editable. `paid` items excluded so the total reads as
  money still to spend.

## 5. Gotchas that cost real debugging time

1. **Time strings crossing noon need an explicit AM.** `"10:40–12:15 PM"` parses
   the start as 10:40 **PM** — the first half inherits AM/PM from the second.
   Write `"10:40 AM–12:15 PM"`. This threw a whole afternoon into the small hours.
2. **`[hidden]` loses to a `display` rule.** Any class with `display:flex|block`
   needs an explicit `.thing[hidden]{display:none}` or it renders anyway.
3. **Icons: use the set, don't draw.** All 34 glyphs are real **Lucide v1.27.0**
   paths, inlined as an SVG sprite (ISC licence, credited in an HTML comment).
   `ico()` emits `class="ic"` which carries baseline sizing — without it a
   missing rule renders a 300px glyph. Fetch more from
   `https://unpkg.com/lucide-static@1.27.0/icons/<name>.svg`.
4. **SVG elements have no `offsetParent`.** Don't use it to test icon
   visibility; it silently passes everything.
5. **The browser preview pane serves stale snapshots** on `file://`. Force a
   reload, and don't trust a measurement taken immediately after a `setDay()` —
   `drawMap()` defers via rAF.
6. **Geolocation doesn't work on `file://`.** Test it on the live URL.

## 6. Standing rules from Maris

These were stated explicitly. Don't relitigate them.

- **Only suggest places from their list.** Everything in `POOL` traces to the
  original Downloads file, their Google Maps flagged pins, or something they
  named. Do **not** add restaurants because they seem good — this happened once
  (Becco, Joe Allen, Sardi's) and was reversed. Verify with:
  `grep -F "<name>" ~/Downloads/nyc-camp-trip-planner\ \(2\).html`
- **No buses.** Subway and rail only. The two camp coaches are a charter, not a
  routing choice, and are labelled as such.
- **No day starts before 9:00 AM.**
- **Monday night ends by 10:00 PM** — early flight Tuesday.
- **Don't invent facts.** Prices, hours and durations get verified or flagged as
  unverified. Several rows carry explicit "confirm this" notes.

## 7. State of the trip

**Booked:** Delta DL 1406/1643 (conf. HTU79Q) · Romer Hell's Kitchen Sept 1–4 ·
Hilton Garden Inn 237 W 54th Sept 7–8 · Stranger Things · Squadron (Groupon
$257.69, prepaid, 2 hours confirmed).

**Budget:** ~$124.50 pp logged (transit + fares; most meals still blank).

**Open decisions**, all in the Checklist tab:
- Squadron slot — **10:00 AM** is the plan; 12:30 / 3:00 / 5:30 also exist.
- Beetle House — **bar walk-in only**. A table means a $65/pp prix fixe charged
  on 24h cancellation. Thursday is the only day it's open during the trip.
- Thursday's downtown afternoon (9/11 Memorial, One World Observatory / Staten
  Island Ferry) is marked **OPTIONAL** — neither was picked by them.
- **Little Island timed passes** — they have used free timed entry for
  late-afternoon arrivals in peak season. The 4:20 PM Wednesday slot is
  deliberately on the safe side of it, but it is **unverified** for Sept 2026.
- All meal slots are unpicked.

### Wednesday was reshaped on 2026-07-27 — Empire State is out

Scott said he didn't need to do the Empire State Building, so it was **cut from
Wednesday** and the freed block went to the **west-side combo that had been cut
for geography** — a trade Maris chose from a list of five on-list options.

Wednesday afternoon is now: one R/W ride to 23rd St → Harry Potter (moved up from
3:35 to open the afternoon) → walk 23rd St west → High Line south → Little Island
→ Chelsea Market → **walk into the Village**, arriving on foot for the 6:30 dinner
and the 8:45 Cellar.

Three consequences that were accepted deliberately — don't "fix" them:

1. **No hotel stop between 9:45 AM and after midnight.** The old 5:00 PM clean-up
   block is gone; the day now ends downtown-to-Village on foot instead of doing a
   Midtown round trip. Maris was told this before choosing.
2. **Walking went up, ~5 mi → ~6.5 mi**, and almost all of the increase is after
   1:45 PM. This is the real cost of the option.
3. **A ~40-minute gap sits between the 5:50 PM Village arrival and 6:30 dinner.**
   That's the `food` "not before" floor holding dinner, working as designed. It's
   genuine slack, not a scheduling bug.

ESB stays in `POOL` (not `dead`), re-addable via **Add to day**. Its checklist
item was deleted and replaced with the Little Island timed-pass check.

**Known gap:** there is **no undo**. Per-day `Reset` clears that day's order,
pins and durations — all or nothing. Worth building if it bites.
