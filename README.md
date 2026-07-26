# NYC + Camp trip planner

A single self-contained HTML page for one trip: New York and Camp No Counselors,
September 1–8 2026.

No build step, no dependencies, no server. Open `index.html` and it works —
including offline, apart from the map tiles.

## What it does

- **Plans** — all eight days as collapsible blocks. Booked things (flights,
  ticketed entries, hotel check-ins, the camp coaches) are anchors with a fixed
  start; everything else floats and reflows around them. Change a duration or
  pin a new time and the rest of the day recalculates. Overruns against a booked
  item are flagged in red.
- **Open slots** — meals with no reservation render as "pick a spot" and offer
  candidates from the saved list, filtered by meal type and ranked by walking
  distance. Picking one renames the row, moves its map pin, and updates the
  directions link on the transit leg either side.
- **Map** — pan, pinch/wheel zoom, geolocation, per-pin walking directions.
  Numbered pins match the numbers on the cards. Flights and transit legs are
  deliberately excluded so the map shows places, not lines.
- **Budget** — per-person costs, editable. Prepaid items are excluded so the
  total reads as money still to spend.
- **Checklist** — everything still needing a booking or a decision.

State (picks, durations, notes, costs, checkboxes) is stored in `localStorage`,
so it is per-browser and does not sync between people.

## Deploying

GitHub Pages serves `index.html` from the repo root. Any push to `main`
redeploys. The page carries a `noindex` tag — it is unlisted, not private.

## Editing

Everything lives in `index.html`. The trip data is the `DAYS` array; the
candidate list is `POOL`. Both are plain JavaScript objects near the top of the
`<script>` block.
