# TradingView MCP — Market Research Assistant

## Context

This project connects Claude Code to TradingView Desktop via Chrome DevTools Protocol (CDP).
TradingView must be running with `--remote-debugging-port=9222` before any commands are used.

To launch TradingView with the debug port:
```
/Applications/TradingView.app/Contents/MacOS/TradingView --remote-debugging-port=9222 &
```

Verify the connection is working before running any command:
```
tv_health_check
```

## Important rules

- Only use TradingView MCP tools to collect data — never fetch from external websites or APIs
- To navigate, use `ui_evaluate` to run `window.location.href = '<url>'` (the CDP target is the
  live tradingview.com page inside TradingView Desktop, so this is same-origin, in-app navigation —
  not an external fetch), then wait 4 seconds for the page to load before reading
- After navigating, use `ui_evaluate` again to extract data from the DOM
- Save progress after each step so partial data is never lost if interrupted
- All output files go to ~/Desktop/market_reports/ — create this folder if it doesn't exist
- If a page takes more than 30 seconds to load, skip it, note it, and move to the next

---

## Commands

### run monthly report

Convenience command that runs both pipeline steps in sequence.

1. Run `market data` to completion (all 10 sectors scraped and saved to JSON)
2. Run `market report` to confirm the dashboard is ready to view

If today's data file (`data_YYYY_MM_DD.json`) already exists with all 10 sectors,
skip straight to `market report`.

---

### market data

Scrapes the top 10 US sectors and saves structured JSON to disk.
This command only collects and saves data — it does not generate any report.
Each run creates a new daily file (`data_YYYY_MM_DD.json`). Only the latest 4 files are kept.

**OUTPUT DISCIPLINE — read this before starting:**
After saving each sector checkpoint, print exactly one confirmation line:
```
✓ [Sector Name] — 20 stocks saved ([N]/10 complete)
```
Do NOT print stock tables, formatted sector summaries, or raw JSON to the conversation.
The data is already on disk. Verbose output fills the context window and causes compaction.

---

**Step 1 — Verify connection**

Use tv_health_check to confirm TradingView is connected.
If it fails, stop and tell the user to launch TradingView with the debug port:
```
/Applications/TradingView.app/Contents/MacOS/TradingView --remote-debugging-port=9222 &
```

---

**Step 2 — Create output folder**

Create ~/Desktop/market_reports/ if it doesn't exist.

---

**Step 3 — Check for today's file and decide whether to scrape**

The target filename for today is: `~/Desktop/market_reports/data_YYYY_MM_DD.json`
(e.g. `data_2026_06_20.json` for June 20 2026 — substitute the actual current date.)

Check whether today's file exists:

**If today's file exists and has all 10 sectors complete:**
- Skip scraping entirely
- Report to the user: "Today's data already exists — data_YYYY_MM_DD.json (10/10 sectors complete)"
- Proceed to Step 7 to verify and confirm

**If today's file exists but is incomplete (partial run):**
1. Read it with the Read tool
2. Check which sector keys are already present under `"sectors"`
3. Note which of the top 10 sectors still need to be collected
4. In Step 6, skip already-completed sectors — do not re-scrape them
5. Report to the user: "Resuming today's file — sectors X, Y, Z already complete"

**If today's file does not exist:**
- Proceed normally from Step 4, writing all data to the new daily file

---

**Step 4 — Collect sector overview data**

Navigate to the TradingView sectors overview page using `ui_evaluate` with
`window.location.href = 'https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/'`.

Wait 4 seconds, then extract the sectors table using `ui_evaluate` with this JavaScript:

```javascript
(() => {
  const rows = document.querySelectorAll('tr');
  const data = [];
  for (let i = 1; i < rows.length; i++) {
    const cells = Array.from(rows[i].querySelectorAll('td'));
    const values = cells.map(c => c.textContent.trim()).filter(Boolean);
    if (values.length >= 3) data.push(values);
  }
  return JSON.stringify(data);
})()
```

This returns all 20 sectors. Take the first 10 rows — these are the top 10 sectors by
market cap since TradingView sorts by market cap by default.

Save the sector list with their names, market caps, div yields, change %, and volume.
Do NOT print the sector list to the conversation — it will be saved to disk in Step 5.

---

**Step 5 — Map sector names to TradingView URLs**

For each of the top 10 sectors returned, match the sector name to its TradingView URL
using this mapping. If a sector from the page is not in this list, construct the URL by
converting the sector name to lowercase with hyphens
(e.g. "Producer Manufacturing" → "producer-manufacturing"):

| Sector | URL |
|--------|-----|
| Electronic Technology | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/electronic-technology/ |
| Technology Services | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/technology-services/ |
| Finance | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/finance/ |
| Health Technology | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/health-technology/ |
| Retail Trade | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/retail-trade/ |
| Producer Manufacturing | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/producer-manufacturing/ |
| Energy Minerals | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/energy-minerals/ |
| Consumer Non-Durables | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/consumer-non-durables/ |
| Utilities | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/utilities/ |
| Consumer Durables | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/consumer-durables/ |
| Non-Energy Minerals | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/non-energy-minerals/ |
| Commercial Services | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/commercial-services/ |
| Process Industries | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/process-industries/ |
| Health Services | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/health-services/ |
| Transportation | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/transportation/ |
| Industrial Services | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/industrial-services/ |
| Distribution Services | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/distribution-services/ |
| Miscellaneous | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/miscellaneous/ |
| Communications | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/communications/ |
| Consumer Services | https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/consumer-services/ |

---

**Step 6 — Collect top 20 stocks for each of the top 10 sectors**

For each sector URL, do the following:

1. Navigate to the sector URL using `ui_evaluate` with `window.location.href = '<sector URL>'`
2. Wait 4 seconds for the page to load
3. Confirm the page has loaded by checking the page title with `ui_evaluate`:
   ```javascript
   document.title
   ```
4. Extract the top 20 stock rows using `ui_evaluate` with this JavaScript.
   Symbol is in `a[class*="tickerName-"]` and name is in `a[class*="tickerDescription-"]`.
   Use `document.querySelectorAll('tr')` starting from index 1 (skip header row).
   ```javascript
   (() => {
     const rows = document.querySelectorAll('tr');
     const data = [];
     let rank = 0;
     for (let i = 1; i < rows.length; i++) {
       if (rank >= 20) break;
       const cells = Array.from(rows[i].querySelectorAll('td'));
       if (cells.length < 3) continue;
       rank++;
       const fc = cells[0];
       const symbol = fc.querySelector('a[class*="tickerName-"]')?.textContent?.trim() ?? "";
       const name   = fc.querySelector('a[class*="tickerDescription-"]')?.textContent?.trim() ?? "";
       data.push({
         rank,
         symbol,
         name,
         marketCap:     cells[1]?.textContent?.trim() ?? "",
         price:         cells[2]?.textContent?.trim() ?? "",
         changePercent: cells[3]?.textContent?.trim() ?? "",
         volume:        cells[4]?.textContent?.trim() ?? "",
         relVol:        cells[5]?.textContent?.trim() ?? "",
         pe:            cells[6]?.textContent?.trim() ?? "",
         eps:           cells[7]?.textContent?.trim() ?? "",
         epsGrowth:     cells[8]?.textContent?.trim() ?? "",
         divYield:      cells[9]?.textContent?.trim() ?? "",
       });
     }
     return JSON.stringify(data);
   })()
   ```

   CRITICAL: The final JSON must have symbol and name as separate string fields.
   Never save them concatenated as a single "nameSymbol" field.

5. After extracting the Overview table, click the "Performance" tab on the same page.
   To click the tab, find the button whose text is exactly "Performance":
   ```javascript
   (() => {
     const all = Array.from(document.querySelectorAll('*'));
     const perf = all.filter(el => el.children.length === 0 && el.textContent.trim() === 'Performance');
     let node = perf[0];
     while (node && !['A','BUTTON'].includes(node.tagName)) node = node.parentElement;
     node?.click();
     return 'clicked';
   })()
   ```
   Then wait 2 seconds and extract performance data:
   ```javascript
   (() => {
     const rows = document.querySelectorAll('tr');
     const data = [];
     let rank = 0;
     for (let i = 1; i < rows.length; i++) {
       if (rank >= 20) break;
       const cells = Array.from(rows[i].querySelectorAll('td'));
       if (cells.length < 5) continue;
       rank++;
       const symbol = rows[i].querySelector('a[class*="tickerName-"]')?.textContent?.trim() ?? "";
       data.push({
         symbol,
         perf1W:  cells[3]?.textContent?.trim() ?? "",
         perf1M:  cells[4]?.textContent?.trim() ?? "",
         perf3M:  cells[5]?.textContent?.trim() ?? "",
         perf6M:  cells[6]?.textContent?.trim() ?? "",
         perfYTD: cells[7]?.textContent?.trim() ?? "",
         perf1Y:  cells[8]?.textContent?.trim() ?? "",
       });
     }
     return JSON.stringify(data);
   })()
   ```

   The Performance tab column order is: Symbol, Price, Chg%, 1W, 1M, 3M, 6M, YTD, 1Y
   (cells[3]=1W, cells[4]=1M, cells[5]=3M, cells[6]=6M, cells[7]=YTD, cells[8]=1Y)

   Match by symbol and add perf1W, perf1M, perf3M, perf6M, perfYTD, perf1Y to each stock.

6. Write the sector data to disk immediately — do not hold it in memory.
   This is the most important step: each sector write is a checkpoint that
   survives context compaction.

   Procedure:
   - If the JSON file does not yet exist: create it now with the full structure
     (reportMonth, generatedAt, sectorOverview, sectors) and this sector as the
     first entry under "sectors".
   - If the JSON file already exists: Read it first (the Write tool requires a prior
     Read), then merge this sector's key into "sectors", then write the file back.
   - After writing, print exactly one line: `✓ [Sector Name] — 20 stocks saved ([N]/10 complete)`
   - Do NOT print the stock data, do NOT print a formatted table, do NOT summarise the stocks.

   CRITICAL — never hold more than one unwritten sector in memory:
   If context compacts mid-run, all data not yet written to disk is permanently lost.
   A file with 6 sectors on disk is recoverable. A file that was never written is not.

The table columns on TradingView sector pages are:
Overview tab: Symbol (separate), Company Name (separate), Market Cap, Price, Change %, Volume, Rel Vol, P/E, EPS dil TTM, EPS dil growth TTM YoY
Performance tab: Symbol, 1W Chg%, 1M Chg%, 3M Chg%, 6M Chg%, 1Y Chg%, YTD Chg%

---

**Step 7 — Verify JSON completeness**

Read `~/Desktop/market_reports/data_YYYY_MM_DD.json` and confirm:
- All 10 sectors are present as keys under `"sectors"`
- Each sector has exactly 20 stocks
- Each stock has both overview fields (marketCap, price, changePercent, volume,
  relVol, pe, eps, epsGrowth, divYield) and performance fields (perf1W, perf1M,
  perf3M, perf6M, perf1Y, perfYTD)
- The `sectorOverview` array has 10 entries

If any sector is missing or has fewer than 20 stocks, re-scrape that sector only
(Steps 6.1–6.6 for that sector), write it to disk, then re-verify.

**Step 7a — Sign-sanity check**

TradingView renders negative percentages with the Unicode minus sign "−" (U+2212),
not a plain ASCII hyphen "-". This has previously caused a downstream parsing bug
(the dashboard silently stripped "−" before converting it to "-", turning every
negative value positive). Catch sign problems here, before they reach the dashboard:

```bash
python3 -c "
import json, glob

path = sorted(glob.glob('/Users/choohuihao/Desktop/market_reports/data_*.json'))[-1]
with open(path, encoding='utf-8') as f:
    d = json.load(f)

def sign(s):
    if not s or s == '—':
        return None
    s = s.replace('−', '-')
    return 1 if s.strip().startswith('+') else (-1 if s.strip().startswith('-') else 0)

problems = []
for ov in d['sectorOverview']:
    key = ov['name'].lower().replace(' ', '-')
    sec = d['sectors'].get(key)
    if not sec:
        continue
    overview_sign = sign(ov['changePercent'])
    stock_signs = [sign(s['changePercent']) for s in sec['stocks'] if sign(s['changePercent']) is not None]
    if not stock_signs:
        continue
    avg_sign = sum(stock_signs) / len(stock_signs)
    # Flag if the sector-level sign disagrees with the majority of its own stocks
    if overview_sign is not None and overview_sign != 0 and (avg_sign * overview_sign) < 0:
        problems.append(f\"{ov['name']}: sectorOverview says {ov['changePercent']} but {sum(1 for x in stock_signs if x>0)}/{len(stock_signs)} stocks are positive\")

if problems:
    print('SIGN MISMATCH FOUND:')
    for p in problems:
        print(' -', p)
else:
    print('Sign check OK — no mismatches between sector-level and stock-level change direction.')
"
```

If a mismatch is reported, stop and investigate before handing the data off to
`market report` — do not silently proceed. This does not fix the dashboard's own
parsing (already fixed in `market-dashboard/src/App.jsx`'s `parseNum`), it only
catches cases where the scraped data itself has a sign problem.

**Step 7b — Prune old files (keep latest 4)**

After a successful full scrape, delete older files so only the 4 most recent daily files remain:
```bash
ls -t ~/Desktop/market_reports/data_*.json | tail -n +5 | xargs rm -f
```

Report to the user:
```
market data complete — data_YYYY_MM_DD.json
  ✓ 10/10 sectors
  ✓ 200/200 stocks
Run `market report` to start the dashboard.
```

Reference JSON format (for the first incremental write in Step 6):
```json
{
  "reportMonth": "May 2026",
  "generatedAt": "2026-05-10T14:32:00Z",
  "sectorOverview": [
    {
      "rank": 1,
      "name": "Electronic Technology",
      "marketCap": "23.57T USD",
      "divYield": "0.42%",
      "changePercent": "+3.27%",
      "volume": "56.97M",
      "industries": 9,
      "stocks": 351
    }
  ],
  "sectors": {
    "electronic-technology": {
      "title": "Electronic Technology",
      "url": "https://...",
      "stocks": [
        {
          "rank": 1,
          "symbol": "NVDA",
          "name": "NVIDIA Corporation",
          "marketCap": "5.23T USD",
          "price": "215.20 USD",
          "changePercent": "+1.75%",
          "volume": "136.42M",
          "relVol": "0.82",
          "pe": "43.90",
          "eps": "4.90 USD",
          "epsGrowth": "+66.75%",
          "perf1W": "+1.20%",
          "perf1M": "+8.50%",
          "perf3M": "+15.30%",
          "perf6M": "+22.10%",
          "perf1Y": "+45.60%",
          "perfYTD": "+18.90%"
        }
      ]
    }
  }
}
```

---

### market report

Launches the interactive React dashboard that reads the completed JSON data file.
Run this after `market data` has finished and the JSON file is verified complete.

This command uses no TradingView tools. The dashboard lives at ~/Desktop/market-dashboard/
and is a React + Vite app that reads ~/Desktop/market_reports/data_YYYY_MM_DD.json automatically.

---

**Step 1 — Confirm the data file exists**

Check that today's `~/Desktop/market_reports/data_YYYY_MM_DD.json` exists and
has all 10 sectors. Use the Bash tool:
```bash
python3 -c "
import json, glob, os
files = sorted(glob.glob('/Users/choohuihao/Desktop/market_reports/data_*.json'), reverse=True)
if not files:
    print('No data files found')
else:
    with open(files[0]) as f:
        d = json.load(f)
    print(f'Latest file: {os.path.basename(files[0])}')
    print(f'{len(d[\"sectors\"])}/10 sectors, generatedAt: {d[\"generatedAt\"]}')
"
```

If the file is missing or incomplete (fewer than 10 sectors), stop and tell the user:
"Run `market data` first to collect this month's sector data."

---

**Step 2 — Start the dashboard**

Tell the user to open a terminal and run:
```bash
cd ~/Desktop/market-dashboard && npm run dev
```

Then open http://localhost:3000 in their browser.

The dashboard starts two processes via concurrently:
- Vite dev server on port 3000 (React UI)
- Node proxy server on port 3001 (serves JSON data + proxies Finnhub news API)

The proxy auto-detects the latest data_YYYY_MM_DD.json file in ~/Desktop/market_reports/
by sorting files and picking the most recent — so no manual path configuration is needed.

---

**Step 3 — Confirm completion**

Tell the user:
- Data file: ~/Desktop/market_reports/data_YYYY_MM_DD.json (today's date, 10 sectors, 200 stocks)
- Dashboard: http://localhost:3000
- Features: 10 sector tabs → click any stock → detail page with TradingView chart link + Finnhub news
- To stop the dashboard: Ctrl+C in the terminal running npm run dev

---

### check connection

Quick check that TradingView is connected and ready.

Use tv_health_check and report:
- Connection status (Connected / Not connected)
- Current symbol visible on screen
- Current timeframe
- Whether the API is available

If not connected, tell the user to run:
```
/Applications/TradingView.app/Contents/MacOS/TradingView --remote-debugging-port=9222 &
```
Then verify again with tv_health_check.

---

### show sector SECTOR_NAME

Navigate to a specific TradingView sector page and show the top 20 companies.

Replace SECTOR_NAME with any sector name (e.g. "show sector semiconductors").

1. Match the name to the URL mapping in Step 5 of `market data`
2. Navigate to the URL using `ui_evaluate` with `window.location.href = '<url>'`
3. Wait 4 seconds
4. Extract the table using the JavaScript in Step 6 of `market data`
5. Display the top 20 as a formatted table in the terminal:
   Rank | Symbol | Company Name | Price | Chg % | Market Cap | P/E

---

### show sector overview

Navigate to the TradingView sectors overview and show a summary of all 20 sectors.

URL: https://www.tradingview.com/markets/stocks-usa/sectorandindustry-sector/

Extract and display: Rank, Sector, Market Cap, Change %, Volume, Number of Stocks

---

## Adding new commands (for future reference)

To add a new command to this file:
1. Open CLAUDE.md in any text editor (TextEdit on Mac, Notepad on Windows)
2. Add a new section under ## Commands following this pattern:

### your command name

Brief description of what this command does.

Step 1 — ...
Step 2 — ...

3. Save the file
4. The next time you launch Claude Code from this folder, the new command is available
5. Just type the command name naturally (e.g. "show sector overview", "market data")

Commands are plain English — write steps the way you would explain a task to a person.
Claude Code reads this file at startup and matches your typed commands to these definitions.
