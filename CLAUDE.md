# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Kalendárium" (formerly "Névnapnaptár") — a Hungarian name-day calendar with national and Catholic holidays, a year overview, `.ics` export, printing, and localStorage-based followed-name reminders. Single file: `index.html`. No package.json, no build step, no dependency beyond two Google Fonts. Everything — data, styles, logic — lives in that one file inside a single `(function(){ "use strict"; ... })()` IIFE.

## Commands

Run locally:

```bash
python3 -m http.server 8000
```

Then open `http://localhost:8000`. There is no lint/build/test command — verify changes in a browser (or Playwright) directly. Note: `python3 -m http.server` does not send a `charset` in its `Content-Type` header, so if `<!doctype html>`/`<meta charset="utf-8">` (top of file) were ever removed, Hungarian accented characters would render as Windows-1252 mojibake under this exact server — this already happened once and was fixed; don't strip those two lines.

Deployment is GitHub Pages serving directly from the `main` branch root — a push to `main` is a push to production. Live URL: `https://menyuswin.github.io/nevnapnaptar/`.

## Architecture

**Name-day data**: `NAMEDAYS` is `{month: [ [names-for-day-1], [names-for-day-2], ... ]}` — array index `day - 1`. Feb 29 has no entry and `getNames()` special-cases it to return `[]`.

**Holiday data — two independent categories, fixed + movable, merged per day**:
- `NATIONAL_FIXED` / `CATHOLIC_FIXED`: plain objects keyed `"month-day"` → label string.
- `NATIONAL_MOVABLE` / `CATHOLIC_MOVABLE`: arrays of `[offsetFromEasterSunday, label]`. `easterSunday(year)` is the Meeus/Jones/Butcher algorithm; `movableMap(year, offsets)` resolves offsets to that year's actual dates and is memoized in `movableCache` (cache key includes which list it is — don't reuse the same key shape for a third movable list without adjusting it).
- `getHolidays(year, m, d)` collects from all four sources, then **merges entries whose label text is identical** (e.g. Dec 25 "Karácsony" is both a national and a Catholic entry — it becomes one chip with two colored dots) and keeps distinct labels separate (e.g. Dec 26 "Karácsony másnapja" vs "Szent István első vértanú" stay as two chips). Any new holiday source should feed into this same merge step, not bypass it.

**The `[hidden]` attribute is not enough on its own here.** Several elements (`.cal-grid`, `.weekrow`, `.selected-strip`, `.year-view`, `.today-alert`) have their own `display: grid|flex` CSS rule, which beats the UA stylesheet's `[hidden] { display: none }` on specificity grounds. There's a single consolidated rule (search for `[hidden]{ display:none !important; }`) that overrides this — if you add a new element that gets toggled via the `hidden` attribute and it doesn't visually disappear, it's missing from that rule, not a JS bug.

**Year view is a mode switch, not a separate page.** `renderCalendar()` (month grid) and `renderYearView()` (chronological list of the year's holidays only, grouped by month, no name-days) are mutually exclusive; `setMode()` toggles which is visible and repoints what `prev-btn`/`next-btn` do (month ± 1 vs. year ± 1). `view` (`{year, month}`) is the single source of truth for both modes and is persisted to `localStorage['kalendarium-view']`.

**Followed names / "reminder" feature is deliberately client-only — no push, no email server.** `followed` (array of exact name strings) and an optional `reminderEmail` string live in `localStorage` (`kalendarium-followed`, `kalendarium-reminder-email`) and never leave the browser. "Sending a reminder" means: if today's date matches a followed name, show `#today-alert` with a `mailto:` link (`buildMailto()`) that opens the visitor's own mail client with a prefilled subject/body — the visitor still has to hit send, and nothing fires unless they have the page open that day. This was a deliberate scope decision (a real push/email-on-schedule system needs a backend, a subscriber database, and a scheduled job — a fundamentally different, hosted project, not a fit for this static repo) — don't quietly "upgrade" this to server-sent notifications without that being an explicit, separately-scoped request.

**Star-toggle UI is event-delegated, not per-element listeners.** Name chips (`nameChipHtml()`) are re-created on every render (hero, selected-day strip, calendar cells all call it), so clicks are caught by a single delegated `document.addEventListener('click', ...)` that matches `.name-chip` / `.fl-remove` via `closest()`. Add new interactive chip-like elements through this same delegation point rather than attaching listeners after each render.

**`.ics` export and printing are both zero-backend, browser-native:** `buildIcs()` generates the iCalendar text for the currently viewed year's holidays and triggers a download via a `Blob` + temporary `<a download>` — no server round-trip. Printing uses a `@media print` block that hides the search box, nav buttons and sidebar cards, leaving just the visible calendar/year-view content; `window.print()` is the only JS involved.

**Emoji choice matters for cross-platform rendering** — the print button intentionally uses `🖨️` (U+1F5A8 + VS16), not the visually-similar but poorly-supported `🖶` (U+1F5B6), which renders as a broken/tofu glyph on several platforms. If adding another icon-only button, check real-world font coverage before picking the codepoint.
