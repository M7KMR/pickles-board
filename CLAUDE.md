# Pickles Board

A stock-picking leaderboard dashboard for a group of 6 friends. Each person picks stocks every ~2 months. The dashboard shows individual stock performance, who picked each one, and a leaderboard ranked by average % return.

> **⚠️ Do not commit any account details to GitHub.** The repo is public. Account numbers, holder names, the raw AJ Bell PDF/CSV exports, and `data.json` must stay gitignored and local. Only commit the derived, non-sensitive data (`data.js`, `index.html`). When in doubt, leave it out.

## Files

- `index.html` — single-file dashboard, all JS inline
- `data.js` — portfolio data as `window.DATA = {...}`, loaded via `<script>` tag (avoids `fetch()` CORS issues on `file://`)
- `data.json` — source of truth for data, gitignored (contains account details)
- `choosers.md` — raw notes mapping stocks to choosers, gitignored
- `Portfolio-Valuation-*.pdf` — AJ Bell portfolio exports, gitignored (filename contains the account ref)

## Choosers

| ID | Name | Emoji |
|----|------|-------|
| ben | Ben | 🦕 |
| baz | Baz | 👶 |
| willis | Willis | 🦧 |
| rick | Rick | 🐦 |
| harps | Harpham | 👴🏻 |
| gammy | Gammy | 🦑 |

## Data shape

`window.DATA` (in `data.js`) has these keys:

- `choosers` — the 6 people (see table above).
- `stocks` — current holdings. Each: `ticker, name, exchange, chooser, quantity, price, currency, value, cost, change, changePct, todayPct`. Optional `formerly` (string) records what a holding used to be before a rename/rebrand/switch (e.g. CNSL `"Omega Diagnostics"`), unused by rendering but preserved as history.
- `exited` — sold positions no longer held. Each: `ticker, name, chooser` only. Rendered in a table at the bottom with `?` for cost/value/change (we don't track exit prices).
- `previous` — the **prior** snapshot, used for the "since last update" features (headline delta, leaderboard rank arrows, Biggest Movers). Shape: `{ asOf, portfolio: { totalValue, totalChangePct }, stocks: [ { ticker, chooser, changePct } ] }`. All comparison UI is gated on `D.previous` existing — remove it and those sections vanish cleanly.
- `portfolio` — `totalValue, totalCost, totalChange, totalChangePct, cash, asOf`.

All rendering reads from `window.DATA`; `index.html` needs no edits on a data update.

## Updating data

Do these **in order** — step 3 is the one that's easy to get wrong.

1. **Export** a new PDF from AJ Bell.
2. **Commit first.** Ensure the *current* `data.js` is committed before editing. This is load-bearing: git history is the source of truth for the `previous` block (see step 3) and preserves the comparison timeline.
3. **Roll `previous` forward — BEFORE changing anything else.** Replace the entire `previous` block with the values from the *outgoing* snapshot:
   - `previous.asOf` = the current (about-to-be-replaced) `portfolio.asOf`.
   - `previous.portfolio` = current `portfolio.totalValue` / `totalChangePct`.
   - `previous.stocks` = one `{ ticker, chooser, changePct }` per current stock.
   - If unsure what the outgoing values were, pull them from git: `git show HEAD:data.js` holds the last committed snapshot. Do NOT skip this — if `previous` isn't rolled, the board compares the new data against a stale period with wrong dates.
4. **Reprice** all holdings in `stocks` from the PDF (price, value, cost, change, changePct, todayPct) and update `portfolio` totals + `asOf`.
5. **Handle roster changes:**
   - New buy → add a `stocks` row; ask the user for the `chooser` if not derivable.
   - Sold holding → move it to `exited` (keep `ticker, name, chooser`).
   - Rename/rebrand → update `name`, add/keep `formerly`.
6. **Sync `data.json`** — it's the gitignored source of truth and must mirror `data.js` exactly (same edits, just without the `window.DATA =` wrapper).
7. **Verify** — the page renders entirely from JS, so a syntax error blanks the whole board. Run the script against the data before shipping:
   ```
   node -e 'global.window={};require("./data.js");console.log("stocks",window.DATA.stocks.length,"exited",window.DATA.exited.length,"prev",window.DATA.previous.asOf)'
   python3 -c 'import json;json.load(open("data.json"));print("json ok")'
   ```
   Watch for temporal-dead-zone bugs: `const p = D.portfolio` is declared partway down the script, so code above it must use `D.portfolio.*`, not `p.*`.
8. **Commit and push** — GitHub Pages updates automatically. Committing each update is what keeps the `previous`/history mechanism working next time.

## Hosting

GitHub Pages: https://m7kmr.github.io/pickles-board/
Repo: https://github.com/M7KMR/pickles-board

## Resolved issues

### iOS mobile: page scroll truncated / fame-shame section caused horizontal overflow (fixed)

The "Hall of Fame & Shame" section overflowed horizontally on narrow screens, and that overflow made iOS miscalculate the page's total scroll height, cutting off the stock list partway down. The two symptoms were one bug — fixing the overflow fixed the scroll truncation.

Root cause: `.fame-shame` used `grid-template-columns: 1fr 1fr`, and `1fr` is shorthand for `minmax(auto, 1fr)`. The `auto` minimum refuses to shrink a track below its content's min-content width, so long stock names + large percentage strings (e.g. `+425.37%`) pushed the grid wider than the viewport. `min-width: 0` on the flex children inside the panels couldn't help because the overflow originated at the grid track level.

Fix (in `index.html`):
- `.fame-shame` tracks changed to `minmax(0, 1fr)` so they can shrink below content min-width. This is the actual fix.
- Added `overflow-x: clip` on `body` as a safety net. Note: `clip` (not `hidden`) is essential — `overflow-x: hidden` turns the element into a scroll container and kills vertical/momentum scroll on iOS Safari, which is why earlier attempts failed. `clip` contains overflow without that side effect.
