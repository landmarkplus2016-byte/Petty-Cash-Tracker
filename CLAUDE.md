# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**LMP PettyCash Tracker** — an Arabic RTL Progressive Web App (PWA) for petty cash tracking. No build step, no server, no dependencies to install. Open `index.html` directly in any browser, or install as a PWA on mobile.

## File Structure

| File | Purpose |
|---|---|
| `index.html` | App shell — HTML structure and all page markup |
| `app.js` | All JavaScript logic — data, rendering, export, voice |
| `styles.css` | All CSS styles |
| `sw.js` | Service worker — caches assets for offline/PWA use |
| `manifest.json` | PWA manifest (name, icons, theme color) |
| `cashflow.png` | PWA icon |
| `LMP Big Logo.jpg` | Source logo (embedded as base64 inside `app.js`) |
| `Report.pdf` | Reference report format the app output must match |
| `Banner-2.jpg` / `New banner-*.png` | Reference design images used to guide banner styling |

## Architecture

The app is a **vanilla JS single-page application** split across `index.html` + `app.js` + `styles.css`. There is no framework, no module bundler, and no backend.

### CDN Libraries (loaded from jsDelivr/Google)
- **Bootstrap 5 RTL** — layout and components
- **Bootstrap Icons** — icon font
- **Cairo** (Google Fonts) — Arabic typeface for all text
- **Chart.js** — loaded but analytics page has been removed; can be dropped in a future cleanup
- **ExcelJS 4.4.0** — Excel export (supports RTL, cell styling, embedded images, dynamic columns)
- **jsPDF + html2canvas** — PDF export (canvas-based, avoids Arabic font issues)

### Data Layer
All data lives in `localStorage` under key `sa_pettycash_v1`:
```js
DB = {
  account: { name, number },   // only 2 fields
  entries: [{ id, date, details, siteId, category, subCategory, custody, settlement, _balance }],
  seq: 1   // auto-increment id counter
}
```
`_balance` is a **computed field** — never stored, always recalculated by `computedEntries()` which sorts entries by date+id and runs a running total. Always call `computedEntries()` after any mutation before calling `persist()`.

**Balance formula:** `balance = previous_balance - custody + settlement`
- عهده (custody) = money OUT → negative, shown in red (`--danger`)
- تسوية (settlement) = money IN → positive, shown in green (`--success`)

**Entry fields:**
- `siteId` — free-text site identifier (e.g. `D1234`). Optional but used for grouping in reports.
- `category` — single-select from fixed list: `انتقالات | أكراميات | أقامة | أخري`. Required on save. Determines which expense column the amount appears in on the Excel export.
- `subCategory` — required only when `category === 'أخري'`. Selected from a fixed list of 16 government-fee/permit items (see below). Stored as empty string `''` for all other categories.

**Backward-compat note:** Old entries may have category values from the previous list (`أنتقالات`, `إكراميات`, `نقل`, `عماله`, `إقامة`). The Excel export normalises these via `CAT_NORM` mapping: `أنتقالات→انتقالات`, `إكراميات→أكراميات`, `إقامة→أقامة`, `نقل→أخري`, `عماله→أخري`.

### Category & SubCategory System

**Main categories** (`#eCategory`):
```
انتقالات | أكراميات | أقامة | أخري
```

**SubCategory** (`#eSubCategoryDiv` / `#eSubCategory`) — visible only when أخري is selected (`onCategoryChange()`):
```
رسوم التبرع | رسوم البيئة | رسوم الزراعة | رسوم الطيران المدني |
رسوم بدلات الطيران المدني | رسوم المعاينة المبدئية | رسوم المقايسة |
رسوم تأمين العداد | رسوم شحن العداد | رسوم متوسط الاستهلاك |
رسوم ترخيص إدارة التفتيش والمتابعة-وزارة الدفاع |
رسوم ترخيص السور - إدارة التفتيش والمتابعة بوزارة الدفاع |
اكراميات الترخيص | اكراميات الكهرباء | تأمين الموقع | غفرة الموقع
```

`saveEntry()` validates that subCategory is chosen when category is أخري. `editEntry(id)` shows/hides `#eSubCategoryDiv` and populates `#eSubCategory`. `resetAddForm()` clears it and hides the div.

### Page Navigation (SPA)
Pages are `<div class="page">` elements toggled with `.active`. Navigate with `goPage(name)` where name is one of: `dashboard`, `add`, `records`, `report`, `settings`. Each page has a corresponding render function called inside `goPage()`.

**Analytics page removed** — the `page-analytics` div, its nav button, and the `renderAnalytics()` call in `goPage()` were all deleted. Chart.js CDN tag remains but is unused.

### Logo Embedding
The logo is stored as `const LOGO_SRC = 'data:image/jpeg;base64,...'` (≈39 KB) near the top of `app.js`. On `DOMContentLoaded`, it is injected into `['settingsLogo']` img elements. In the JS-generated report HTML it is referenced as `${LOGO_SRC}` inside a template literal.

To re-embed the logo after changing `LMP Big Logo.jpg`:
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes('LMP Big Logo.jpg')) | Out-File logo_b64.txt -NoNewline -Encoding ASCII
```
Then replace the base64 string inside `const LOGO_SRC = 'data:image/jpeg;base64,...'` in `app.js`.

### Sticky Header Layout
Both the banner and blue ribbon are wrapped in `<div id="sticky-top">` with `position: sticky; top: 0`. They freeze together on scroll. Neither child has its own sticky/fixed positioning.

### Report Generation & Export

#### PDF report — `buildReport()` + `doExportPDF()`
- `buildReport()` generates a `#reportContent` div with inline styles. The table is wrapped in `overflow-x:auto` for in-app scrolling.
- `doExportPDF()` — temporarily sets `el.style.minWidth = '700px'` before calling `html2canvas` so all columns render fully, then restores. Embeds canvas as image in jsPDF.
- `reportEntries` (set inside `buildReport()`) holds the flat un-grouped individual entries — used by `doExportExcel()` which does its own grouping independently.

**PDF table columns:** م | التاريخ | اسم الموقع | البيان | المصروفات | العهدة | الفرق

**PDF grouping logic** — entries are grouped by **(siteId + البيان)** key before rendering:
- `البيان` = `subCategory` if set, otherwise `details`
- When the same site has both a custody entry AND a settlement entry with the same البيان, they merge into **one row** — custody and settlement are summed, الفرق = custody − settlement
- Entries with no `siteId` are never grouped; they appear individually below the site rows
- The date shown for a merged group is the **latest** date among the merged entries
- Total row at bottom shows sums of المصروفات and العهدة columns

#### Excel export — `doExportExcel()`
The Excel is a **"كشف تسوية عهدة"** form. Key behaviours:

- **Dynamic category columns** — scans `reportEntries` for which of the 4 categories are actually used. Only those columns are generated.
- **Column layout** (1-based, single-letter A-Z):
  - A: م  B: التاريخ  C: اسم الموقع  D: البيان
  - E … (4+N): dynamic expense category columns (انتقالات / أكراميات / أقامة / أخري)
  - No العهدة column. No الفرق column.
- **البيان column** — for grouped site rows shows the first entry's subCategory or details; for individual rows shows subCategory (if أخري) else details.
- **Grouped by siteId** — one row per site, aggregating custody and catAmts. Entries without a siteId appear individually. `colL(n)` helper converts 1-based column number → letter.
- **Header**: `اسم أمين العهدة` pulls from `DB.account.name`; `تاريخ الكشف` is today's date at export time (`DD/MM/YYYY`).
- **المجموع row** — per-column totals for every expense category column.
- **الإجمالي الكلي row** — directly below المجموع. Label (A–D merged) + single merged value (E–lastCol) = sum of ALL category column totals. Styled in dark blue with white bold text.
- **No الرصيد الحالي row** — removed.
- Logo is placed at `tl: { col: logoStart0, row: 0 }` where `logoStart0 = max(totalCols-2, 4)` (0-based).
- Downloads via `wb.xlsx.writeBuffer()` → Blob → `URL.createObjectURL`.

### Favicon
Two `<link>` tags in `<head>` embed a gold-coin SVG as base64 data URI:
```html
<link rel="icon" type="image/svg+xml" href="data:image/svg+xml;base64,...">
<link rel="apple-touch-icon" href="data:image/svg+xml;base64,...">
```
To update: generate new SVG, base64-encode it in PowerShell (`[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($svg))`), replace both hrefs.

### Details Field
The التفاصيل input is a `<textarea rows="3">` (not `<input>`), wrapped in a Bootstrap `input-group` with a mic button. Supports Enter for new lines.
- `.entry-title` CSS has `white-space:pre-line; word-break:break-word; overflow-wrap:break-word;` so multi-line/long text renders correctly in the records list.
- `.rt .detail-td` has the same word-break rules plus `white-space:pre-wrap` for the report table.

### Add Entry Form Fields
The form (`#page-add`) now has these fields in order:
1. نوع العملية — radio toggle: عهدة (custody) / تسوية (settlement)
2. التاريخ — `<input type="date" id="eDate">`
3. رقم الموقع — `<input type="text" id="eSiteId">` (optional, used for report grouping)
4. الفئة — `<select id="eCategory" onchange="onCategoryChange()">` single-select, required. Options: انتقالات / أكراميات / أقامة / أخري
5. التصنيف الفرعي — `<select id="eSubCategory">` inside `<div id="eSubCategoryDiv">`. Hidden unless أخري is selected. Required when visible.
6. التفاصيل — `<textarea id="eDetails">` with mic button
7. المبلغ — `<input type="number" id="eAmount">` with mic button

`saveEntry()` validates date, details, category, subCategory (when أخري), and amount > 0. `resetAddForm()` clears all fields including subCategory and hides `#eSubCategoryDiv`. `editEntry(id)` populates all fields and shows/hides `#eSubCategoryDiv` based on category.

### Edit Entry Pattern
**Critical**: in `editEntry(id)`, call `goPage('add')` FIRST (which triggers `resetAddForm()`), then populate the form fields. Reverse order causes the reset to wipe the populated values.

### Report Header Layout
The report header uses `direction: ltr` on the outer flex container to force **logo LEFT / account info RIGHT**, which is the reverse of the page's RTL default. The inner info div re-applies `direction: rtl` for correct Arabic text rendering.

### Arabic Voice Recognition
Added to the Add Entry form (`page-add`). Uses the browser's built-in **Web Speech API** — no library needed.

- **`startVoice(target)`** — takes `'details'` or `'amount'`. Sets `lang: 'ar-SA'`, shows red pulsing state on the mic button while listening, then routes the transcript to the correct field.
- **`resetMicBtn(btn)`** — restores the button to its default outline state after recognition ends (success, error, or silence).
- **`parseArabicNumber(text)`** — converts spoken Arabic to a number. Tries Eastern Arabic digits (٠-٩) first, then parses words like "مية وخمسين" → `150`. Returns `null` if unrecognized.

**Browser support:** Chrome/Edge = full. Safari = partial (needs HTTPS). Firefox = not supported (shows Arabic warning toast).

**UI pattern:** Both `#eDetails` (textarea) and `#eAmount` (number input) are wrapped in Bootstrap `input-group` divs. The mic button appends to the end of the group; in RTL layout this renders visually to the left of the field.

### PWA / Service Worker
`sw.js` caches `index.html`, `app.js`, `styles.css`, `cashflow.png`, and `manifest.json` for offline use.

**Critical — cache busting:** The `CACHE` constant in `sw.js` (e.g. `lmp-petty-cash-v1.18`) **must be incremented every time any file changes** before pushing to GitHub. The `activate` handler deletes all caches that don't match the current name, so bumping the version forces phones to download fresh files. Version sequence: `v1` → `v1.10` → `v1.11` → … → `v1.18` → ...

## Windows / PowerShell Environment

- Python and Node.js are **not available** — use PowerShell for all file operations.
- **Critical**: Never put Arabic text as string literals inside `.ps1` files — causes parse errors.
- Always read/write HTML files with explicit UTF-8 encoding:
  ```powershell
  $html = [System.IO.File]::ReadAllText("file.html", [System.Text.Encoding]::UTF8)
  [System.IO.File]::WriteAllText("file.html", $html, [System.Text.Encoding]::UTF8)
  ```
- Run scripts: `powershell -ExecutionPolicy Bypass -File script.ps1`
- When doing string replacements on the HTML, use **ASCII-only** search strings where possible (e.g. search for English words, not Arabic).

## Key Design Constraints

- **Multi-file but no build step**: `index.html`, `app.js`, `styles.css` are separate files — do not merge them back into one file.
- **Offline capable**: Service worker caches all local assets. CDN libraries load on first visit and are cached by the browser.
- **RTL throughout**: `dir="rtl"` on `<html>`. Bootstrap RTL CDN is used (not standard Bootstrap).
- **Arabic number-to-words**: `toArabicWords()` — unit comes **before** ten in Arabic grammar (أربعة وأربعون, not أربعون وأربعة).
- **Currency**: All amounts in EGP (ج.م), formatted with `fmt()` → `toLocaleString('en-US', {minimumFractionDigits:2})`.
- **Desktop**: Capped at `max-width: 480px; margin: 0 auto` on screens ≥ 600px.
- **Cache bump required**: Always increment `CACHE` in `sw.js` with every change (see PWA section above).
