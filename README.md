# Table to CSV & XLSX — FocusPlus

A Chrome extension that turns any web table into **CSV, XLSX, JSON, Markdown or clipboard TSV**.
Four permissions, **no host permissions**, **zero network requests**, **500 rows per table**.
Version `1.0.0`. Free, permanently.

- **Homepage:** https://oguenfoude.github.io/focusplus-table/
- **Privacy policy:** https://oguenfoude.github.io/focusplus-table/privacy-policy.html
- **Disclaimer:** https://oguenfoude.github.io/focusplus-table/disclaimer.html
- **Support:** oguenfoude@proton.me

---

## 1. What it is

A local-first table extractor. It reads the tables already rendered on the page you invoked it
on and writes them out in a format a spreadsheet can open. Nothing is uploaded, nothing is
tracked, nothing is stored remotely.

| Property | Value |
|---|---|
| Manifest | V3, `minimum_chrome_version` 102 |
| Permissions | `activeTab`, `storage`, `scripting`, `contextMenus` |
| Host permissions | **none — the key is absent from the manifest** |
| Network requests | **0** |
| Remote code / `eval` / `innerHTML` | **0** |
| Telemetry | **0** |
| Row limit | 500 per table, header included |
| Formats | CSV, XLSX, JSON, Markdown, Copy TSV, Export All → multi-sheet XLSX |
| Price | Free. No licence key, no account, no paywall |

### Why the permission list matters

Chrome shows one install warning per permission. The widest is
*"Read and change all your data on all websites"*, produced by `<all_urls>`. Every significant
competitor in this category requests it. This build does not, so that warning is never shown.
`activeTab` grants access to the current tab only, only after you invoke the extension, and is
revoked when you navigate away.

---

## 2. Using it — every step

### 2.1 Scan a page

1. Open the page that contains the table.
2. Click the FocusPlus toolbar icon — or press **`Ctrl+Shift+Y`**.
3. The popup lists every table on the page. Each card shows:
   - the table's **true size** (`10,001 × 12`) — detection is never capped;
   - a **`500 max`** badge when the table is larger than what an export will contain;
   - its kind (`HTML Table`, `ARIA Grid`, `CSS Grid`, `CSS Flex`, `CSS Table`);
   - a `nested` badge for a table inside another table, and a scope badge (`shadow`, `iframe`);
   - a one-line header preview, rendered right-to-left when the content is RTL.
4. Hover a card to highlight that table on the page.
5. Click a format button to download it.

If the popup is closed and reopened, the last scan is restored for the same page, as long as
*Remember last scan* is ticked.

### 2.2 Pick a table directly on the page

1. Press **`Alt+Shift+T`**, or click **Pick table** in the popup, or right-click → *Pick a table on this page*.
2. Move the pointer. The table under the cursor is outlined and a badge shows its live
   **rows × cols**.
3. Click. The table is captured immediately — already capped — and the popup offers it with
   CSV and XLSX buttons on the next open.
4. **`Esc`** cancels. The overlay, the badge and every listener are removed; the page is left
   exactly as it was.

### 2.3 Clean before exporting

Click **Cleaning** in the popup to open the panel. Every control applies to the next export
and is remembered locally.

| Control | Default | Effect |
|---|---|---|
| **CSV delimiter** | `,` | `,` / `;` / tab. Quoting follows the delimiter — a cell containing `,` is not quoted when the separator is `;`. |
| **Trim & collapse whitespace** | on | Collapses runs of spaces, tabs and newlines inside a cell, then trims the ends. |
| **Strip empty rows** | off | Removes any row whose every cell is empty. |
| **Include header row** | on | Off drops row 0 from the export. |
| **Leading-zero & formula safety** | on | Keeps `004512` as text in every format, and prefixes cells starting with `=` `+` `-` `@` with an apostrophe so a spreadsheet cannot execute them. |
| **Columns** | all | Per-table. Click **Columns** on a card to load its header row, then untick anything you do not want. Every exported row loses the same indices, so the output stays rectangular. |

Cleaning runs **after** the 500-row cap, never before, so the cap can never be bypassed by a
cleaning transform.

### 2.4 Formats

| Format | Notes |
|---|---|
| **CSV** | UTF-8 BOM + CRLF, so Excel opens it without the import wizard. RFC-4180 quoting, delimiter-aware. |
| **XLSX** | Written off-thread in a Web Worker. Every cell is typed as text, so leading zeros survive and no formula is ever written. Column widths auto-fit. |
| **JSON** | Array of objects keyed by the header row, duplicate headers disambiguated (`Name`, `Name_2`), pretty-printed. |
| **Markdown** | GitHub-flavoured table with a header separator. Pipes escaped, newlines become `<br>`. |
| **Copy TSV** | Tab-separated to the clipboard, for pasting straight into Sheets or Excel. |
| **Export All (.xlsx)** | Every table on the page as its own worksheet in one workbook. Each sheet is independently capped. |

### 2.5 Right-click and keyboard

| Action | Shortcut | Notes |
|---|---|---|
| Open popup | `Ctrl+Shift+Y` (`Cmd+Shift+Y` on macOS) | `commands._execute_action`, not a permission |
| Pick a table | `Alt+Shift+T` | `commands.pick-table` |
| Export first table to CSV | right-click → *Export table to CSV* | Uses the same capped path |
| Pick a table | right-click → *Pick a table on this page* | Same picker as the shortcut |

### 2.6 What it cannot read

These are browser limits, documented in full on the
[disclaimer page](https://oguenfoude.github.io/focusplus-table/disclaimer.html):

- `chrome://` pages and the Chrome Web Store — Chrome forbids all injection there.
- `<canvas>`-rendered data — pixels, not text nodes.
- Cross-origin iframes — blocked by the same-origin policy, and this build requests no host permission to cross it.
- Closed shadow roots — no `shadowRoot` reference exists.
- Infinite scroll and pagination — only what is currently in the DOM is exported.
- Hover-only or click-revealed text — only rendered, visible text is read.

---

## 3. Architecture

No framework, no build step, no bundler. Plain ES5-compatible scripts with UMD wrappers, so
the same file loads in a page, in a service worker, and under Node for unit tests.

```
extension-core/
  manifest.json
  src/
    core/
      detect.js        detection: HTML tables, ARIA grids, CSS grid/flex, shadow DOM, iframes
      extract.js       DOM -> rectangular matrix; span expansion, text extraction, cap seam
      rowcap.js        stateless 500-row cap (MAX_ROWS, apply, notice)
      clean.js         pre-export cleaning pipeline (columns, trim, header, strip-empty)
      session-store.js last-scan persistence with tab/url/age validation
    content/
      inject.js        page bridge in the isolated world: detect, extract, highlight, picker
    background/
      sw.js            message router, injection, defence-in-depth cap, context menu, commands
    popup/
      popup.html/.css/.js   UI, cleaning panel, column chips, picked banner
      xlsx-worker.js        off-thread XLSX writer
    export/
      csv.js tsv.js json.js markdown.js xlsx.js
  vendor/xlsx.full.min.js   SheetJS CE 0.18.5 (Apache-2.0)
  assets/icon-{16,48,128}.png
```

### The cap is applied once, at one seam

`extract.js` materialises the full grid, expands `rowspan`/`colspan`, pads every row to the
widest column, and only then calls `rowcap.apply`. Capping earlier would break
`rowspan="0"` (which resolves against the end of the row group) and leave the padding loop
reading rows that no longer exist.

`totalRows` is passed to `rowcap.apply` **explicitly**. Deriving it from `matrix.length` after
the cut would report `500` as the true size and hide the truncation from the user.

`sw.js` re-applies the cap as defence in depth and preserves the page-reported `totalRows`.

### Performance

Extraction of a 10,001 × 12 table (120,012 cells) went from **2,070 ms to ~133 ms** — a 15×
improvement — with no change in output. The cost was almost entirely `getComputedStyle`:

| Fix | Effect |
|---|---|
| One per-pass visibility/style cache (`createCache`) | Removed the duplicate `isHidden` call per cell |
| `cleanText` fast path behind one combined probe regex | Skipped ~480,000 regex operations on already-clean cells |
| `spanInfo()` — one attribute read per attribute | Replaced ~600,000 `getAttribute` calls with ~480,000 |
| Single-text-node fast path in `visibleText` | Avoided a recursive walk for the common cell |
| Rows past the limit are not materialised | Skipped text extraction for 9,501 of 10,001 rows |

### Anti-bypass posture

The cap is a speed bump, not a vault. Five entry points can produce a matrix, and all five
pass through `rowcap.apply`:

| Entry point | Capped by |
|---|---|
| Popup → `EXTRACT` | page bridge + `enforceCap()` |
| Popup → `PREVIEW` (20 rows) | `enforceCap(value, 20)` |
| Popup → `EXTRACT_ALL` (per sheet) | per-table `EXTRACT` |
| Context menu → first table | `enforceCap()` |
| Picker → `PICKED` | `enforceCap()` before storage |
| DevTools → `__TTC__.extract(id)` | page-side default limit 500 |

A page cannot spoof results: MV3 isolated worlds give the content script its own
`globalThis`, verified by a test that plants a hostile `window.__TTC__` in the main world.
A client-supplied `limit` field is ignored by the service worker.

---

## 4. Tests

All suites must be green before any package is cut.

```bash
cd /home/focus/ext-research-workspace
export HOME=/home/focus
export PLAYWRIGHT_BROWSERS_PATH=/home/focus/.cache/ms-playwright

# Unit — 63 checks (core engine, exporters, session store, row cap, cleaning)
node --test tests/unit/core.test.mjs tests/unit/export.test.mjs \
           tests/unit/session.test.mjs tests/unit/rowcap.test.mjs \
           tests/unit/clean.test.mjs

# Baseline E2E — 39 checks (detection, all formats, popup, restore)
python3 tests/e2e/run.py

# UI + 500-row cap, byte-level — 39 checks
python3 tests/e2e/free_ui.py

# Picker, cleaning, anti-bypass, responsive — 40 checks
python3 tests/e2e/focusplus_ui.py

# Torture — 34 checks, includes the <800 ms performance gate
python3 tests/e2e/torture_sweep.py

# Live sites — 38 checks (w3schools, Wikipedia, W3C, restricted pages)
python3 tests/e2e/live_sweep.py
```

**Total: 253 checks.**

### The gates that are not negotiable

| Gate | Assertion |
|---|---|
| Performance | `10,001 × 12` extraction **< 800 ms** (measured ~133 ms) |
| Cap | downloaded CSV has **exactly 500 records**, XLSX has **exactly 500 rows** |
| Honesty | status reads `first 500 of 1,001 rows` |
| Bytes | UTF-8 BOM present, leading zeros preserved, `=SUM(A1)` inert |
| Privacy | **zero** external network requests during the whole suite |
| Picker | `Esc` leaves **zero** overlay nodes and no active picker |
| Anti-bypass | direct message, picker, context menu, tampered bridge all cap at 500 while keeping `totalRows` |
| Responsive | no horizontal overflow at 390 px |
| Console | zero uncaught exceptions, zero page errors |

---

## 5. Chrome Web Store submission sheet

Paste-ready. Everything below matches the shipped `manifest.json` and the live pages.

### Store listing

| Field | Value |
|---|---|
| **Name** | `Table to CSV & XLSX — FocusPlus` |
| **Short name** | `FocusPlus Table` |
| **Summary (132 char max)** | `Export any web table to CSV, XLSX, JSON, or Markdown. Click-to-select, clean data, 100% local, up to 500 rows/table.` |
| **Category** | Productivity → Developer Tools *(choose one; Productivity is the better fit)* |
| **Language** | English |
| **Homepage URL** | `https://oguenfoude.github.io/focusplus-table/` |
| **Support URL** | `https://oguenfoude.github.io/focusplus-table/` |
| **Privacy policy URL** | `https://oguenfoude.github.io/focusplus-table/privacy-policy.html` |

### Detailed description

```
Turn any table on any page into a spreadsheet file — without giving the extension access to
every site you visit.

WHAT IT DOES
• Scan the current page and list every table it finds, with its true size.
• Click-to-select picker: press Alt+Shift+T, hover, click the table you want.
• Export to CSV, XLSX, JSON or Markdown. Copy as TSV for pasting into Sheets or Excel.
• Export every table on the page as one multi-sheet workbook.

CLEAN BEFORE YOU EXPORT
• CSV delimiter: comma, semicolon or tab.
• Trim and collapse whitespace.
• Strip empty rows.
• Include or drop the header row.
• Choose which columns to keep, per table.
• Leading-zero and formula safety, on by default.

BUILT FOR AWKWARD TABLES
• Merged cells, rowspan="0" and oversized colspan handled without crashing.
• Nested tables, ARIA grids, CSS grids, shadow DOM and same-origin frames.
• Right-to-left content keeps its correct reading order.
• 004512 stays 004512. =SUM(A1) cannot execute when the file opens.

PRIVACY
• Zero network requests. No telemetry, no analytics, no account.
• Four permissions: activeTab, storage, scripting, contextMenus.
• NO host permissions at all — no "read and change all your data on all websites" warning.
• Everything happens inside your browser. Nothing is uploaded.

THE LIMIT, STATED PLAINLY
Each table exports its first 500 rows, and the header counts as one. The popup always shows
the table's real size next to a "500 max" badge before you export. No time limit, no daily
quota, no account, and nothing to buy.

WHAT IT CANNOT READ
chrome:// pages, data painted into a <canvas>, cross-origin iframes, closed shadow roots, and
rows that a site has not rendered yet. These are browser limits, not bugs — details on the
disclaimer page.

Free, permanently. No licence key. Runs on Chrome 102 and newer, plus Edge and Brave.
```

### Privacy practices (dashboard answers)

| Question | Answer |
|---|---|
| Single purpose description | `Export the tables on the current web page to a local CSV, XLSX, JSON or Markdown file.` |
| Does it collect personally identifiable information? | **No** |
| Does it collect health information? | **No** |
| Does it collect financial and payment information? | **No** |
| Does it collect authentication information? | **No** |
| Does it collect personal communications? | **No** |
| Does it collect location? | **No** |
| Does it collect web history? | **No** |
| Does it collect user activity? | **No** |
| Does it collect website content? | **No** — read transiently to produce a file the user asked for; never transmitted or retained |
| **Are you using remote code?** | **No** |
| **Contains in-app purchases / payments?** | **No** |
| **EU DSA trader status** | **Non-Trader** — no payment is collected by any means |

**Certification:** all three required checkboxes — *not being sold to third parties*, *not being
used for purposes unrelated to the single purpose*, *not being used to determine
creditworthiness* — apply and can be certified truthfully.

### Permission justification (shown during review)

| Permission | Justification |
|---|---|
| `activeTab` | Read the table on the page the user explicitly invoked the extension on. Revoked on navigation. No host permissions are requested, so no broader access exists. |
| `storage` | Persist the user's cleaning preferences and, optionally, the last scan, in `chrome.storage.local` on the user's device only. |
| `scripting` | Inject the extraction script into the activated tab. This is the only mechanism available under MV3 without a persistent content script. |
| `contextMenus` | Provide the right-click "Export table to CSV" and "Pick a table on this page" entries. |

### Assets to upload

| Asset | File |
|---|---|
| Screenshot 1 — scan + 500 max badge | `store-assets/screenshot-1-scanned.png` (1280×800) |
| Screenshot 2 — picker + cleaning panel | `store-assets/screenshot-2-picker-clean.png` (1280×800) |
| Screenshot 3 — RTL, leading zeros, formats | `store-assets/screenshot-3-rtl-excel.png` (1280×800) |
| Screenshot 4 — four permissions vs `<all_urls>` | `store-assets/screenshot-4-privacy.png` (1280×800) |
| Small promo tile | `store-assets/promo-tile-440x280.png` (440×280) |
| Store icon | `extension-core/assets/icon-128.png` |

---

## 6. Building the package

```bash
cd /home/focus/ext-research-workspace
rm -f focusplus-table-1.0.0-release.zip
(cd extension-core && zip -r -X ../focusplus-table-1.0.0-release.zip . \
   -x '*.DS_Store' -x '__MACOSX/*')
sha256sum focusplus-table-1.0.0-release.zip
unzip -l focusplus-table-1.0.0-release.zip | tail -3
```

The archive contains exactly the contents of `extension-core/` with `manifest.json` at the
root. No `tests/`, no `docs/`, no hidden files.

### Pre-flight checks

```bash
# zero stale strings across the shipped tree and the public site
# (search for the retired brand name and the abandoned payment providers)
git --version >/dev/null 2>&1
for pat in "$(printf '\x74\x61\x62\x6c\x6f\x6f\x6d')" polar gumroad whop "no row limit" \
           YOUR_DOMAIN YOUR_SUPPORT_EMAIL REPLACE_WITH; do
  printf '%-22s %s\n' "$pat" "$(grep -rin -- "$pat" extension-core/ docs/ | wc -l)"
done
# expected: every count is 0

# the manifest contract
python3 -c "import json;m=json.load(open('extension-core/manifest.json'));\
print(m['name'],m['version']);print(m['permissions']);\
print('host_permissions' in m, list(m.get('commands',{})))"
```

---

## 7. Licence

Original code: MIT.
SheetJS Community Edition, vendored at `vendor/xlsx.full.min.js`, is Apache-2.0 — see
`vendor/LICENSE.SheetJS.txt` and `THIRD_PARTY.md`.
