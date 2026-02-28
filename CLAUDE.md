# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**LMP PettyCash Tracker** — a single-file Arabic RTL web app for petty cash tracking. No build step, no server, no dependencies to install. Open `LMP PettyCash Tracker Ver 1.html` directly in any browser.

## File Structure

| File | Purpose |
|---|---|
| `LMP PettyCash Tracker Ver 1.html` | The entire app — HTML, CSS, JS, and base64 logo all inline |
| `LMP Big Logo.jpg` | Source logo (already embedded as base64 inside the HTML) |
| `Report.pdf` | Reference report format the app output must match |
| `Banner-2.jpg` / `New banner-*.png` | Reference design images used to guide banner styling |

## Architecture

The app is a **vanilla JS single-page application** inside one HTML file. There is no framework, no module bundler, and no backend.

### CDN Libraries (loaded from jsDelivr/Google)
- **Bootstrap 5 RTL** — layout and components
- **Bootstrap Icons** — icon font
- **Cairo** (Google Fonts) — Arabic typeface for all text
- **Chart.js** — analytics charts
- **ExcelJS 4.4.0** — Excel export (replaces SheetJS; supports RTL, cell styling, embedded images)
- **jsPDF + html2canvas** — PDF export (canvas-based, avoids Arabic font issues)

### Data Layer
All data lives in `localStorage` under key `sa_pettycash_v1`:
```js
DB = {
  account: { name, number },   // only 2 fields
  entries: [{ id, date, details, custody, settlement, _balance }],
  seq: 1   // auto-increment id counter
}
```
`_balance` is a **computed field** — never stored, always recalculated by `computedEntries()` which sorts entries by date+id and runs a running total. Always call `computedEntries()` after any mutation before calling `persist()`.

**Balance formula:** `balance = previous_balance - custody + settlement`
- عهده (custody) = money OUT → negative, shown in red (`--danger`)
- تسوية (settlement) = money IN → positive, shown in green (`--success`)

### Page Navigation (SPA)
Pages are `<div class="page">` elements toggled with `.active`. Navigate with `goPage(name)` where name is one of: `dashboard`, `add`, `records`, `report`, `analytics`, `settings`. Each page has a corresponding render function called inside `goPage()`.

### Logo Embedding
The logo is stored as `const LOGO_SRC = 'data:image/jpeg;base64,...'` (≈39 KB) near the top of the `<script>` block. On `DOMContentLoaded`, it is injected into `['subBannerLogo', 'settingsLogo']` img elements. In the JS-generated report HTML it is referenced as `${LOGO_SRC}` inside a template literal.

To re-embed the logo after changing `LMP Big Logo.jpg`:
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes('LMP Big Logo.jpg')) | Out-File logo_b64.txt -NoNewline -Encoding ASCII
```
Then replace the base64 string inside `const LOGO_SRC = 'data:image/jpeg;base64,...'`.

### Sticky Header Layout
Both the banner and blue ribbon are wrapped in `<div id="sticky-top">` with `position: sticky; top: 0`. They freeze together on scroll. Neither child has its own sticky/fixed positioning.

### Report Generation & Export
- `buildReport()` generates a `#reportContent` div with inline styles (RTL Arabic, LTR flex for logo-left / info-right layout). The table is wrapped in `overflow-x:auto` for in-app scrolling.
- `doExportPDF()` — temporarily sets `el.style.minWidth = '700px'` before calling `html2canvas` so all columns render fully, then restores. Embeds canvas as image in jsPDF.
- `doExportExcel()` — **async**, uses ExcelJS. RTL worksheet, embedded logo (from `LOGO_SRC`), blue header row, alternating row colors, red/green amount coloring, totals row, balance footer. Downloads via `wb.xlsx.writeBuffer()` → Blob → `URL.createObjectURL`.

### Favicon
Two `<link>` tags in `<head>` embed a gold-coin SVG as base64 data URI:
```html
<link rel="icon" type="image/svg+xml" href="data:image/svg+xml;base64,...">
<link rel="apple-touch-icon" href="data:image/svg+xml;base64,...">
```
To update: generate new SVG, base64-encode it in PowerShell (`[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($svg))`), replace both hrefs.

### Details Field
The التفاصيل input is a `<textarea rows="3">` (not `<input>`). Supports Enter for new lines.
- `.entry-title` CSS has `white-space:pre-line; word-break:break-word; overflow-wrap:break-word;` so multi-line/long text renders correctly in the records list.
- `.rt .detail-td` has the same word-break rules plus `white-space:pre-wrap` for the report table.

### Edit Entry Pattern
**Critical**: in `editEntry(id)`, call `goPage('add')` FIRST (which triggers `resetAddForm()`), then populate the form fields. Reverse order causes the reset to wipe the populated values.

### Report Header Layout
The report header uses `direction: ltr` on the outer flex container to force **logo LEFT / account info RIGHT**, which is the reverse of the page's RTL default. The inner info div re-applies `direction: rtl` for correct Arabic text rendering.

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

- **Single file**: Everything must remain inline — no external `.js`, `.css`, or image files.
- **Offline capable**: All assets either CDN-loaded (first visit) or base64-embedded.
- **RTL throughout**: `dir="rtl"` on `<html>`. Bootstrap RTL CDN is used (not standard Bootstrap).
- **Arabic number-to-words**: `toArabicWords()` — unit comes **before** ten in Arabic grammar (أربعة وأربعون, not أربعون وأربعة).
- **Currency**: All amounts in EGP (ج.م), formatted with `fmt()` → `toLocaleString('en-US', {minimumFractionDigits:2})`.
- **Desktop**: Capped at `max-width: 480px; margin: 0 auto` on screens ≥ 600px.
