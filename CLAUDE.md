# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page personal daily-routine tracker ("Thomas's Routine"). Everything — markup, styles, data, and logic — lives in `index.html`. There is no framework, no build step, no package manager, no test suite, and no server component. State persists in the browser via `localStorage`.

The only other tracked file is `icon.png` (used by the `apple-mobile-web-app-*` meta tags so the page can be added to the iOS home screen).

## Running / previewing

Open `index.html` directly in a browser, or serve the directory over any static HTTP server, e.g.:

```bash
python3 -m http.server 8000   # then visit http://localhost:8000/
```

There is nothing to build, lint, or test. Changes are validated by reloading the page.

## Architecture

### Tabs and renderers

Five tabs are switched via `switchTab(tab)` (index.html:533). Each tab has a `<div class="tab-content" id="tab-<name>">` and a dedicated renderer:

| Tab       | Renderer            | Data source                  |
|-----------|---------------------|------------------------------|
| `today`   | `renderTimeline`    | `DAYS[currentDay].blocks`    |
| `clients` | `renderClients`     | `CLIENTS` + `CLIENT_TASKS_*` |
| `habits`  | `renderHabits`      | `HABITS`                     |
| `week`    | `renderWeek`        | `WEEK_EVENTS`                |
| `goals`   | `loadGoals`/`saveGoals` | textareas → state        |

`init()` runs an IIFE on load: it calls `loadState()`, paints the date header, restores supplement pills, and calls `setDay(todayKey())` to render today's schedule.

### Hardcoded data constants

All content is defined as JS object literals near the top of the `<script>` block (index.html:290–479). Editing the routine = editing these constants:

- `CLIENTS` — id, display name, and avatar color for each dating-VA client.
- `CLIENT_TASKS_AM` / `CLIENT_TASKS_PM` — the 4 morning + 4 evening tasks applied to every client.
- `DAYS` — keyed by `mon`…`sun`. Each value has `label` and `blocks[]`. Each block has `time`, `end`, `title`, `sub`, `type`, `tag`.
- `HABITS` — keyed by category (`morning`, `work`, `body`, `evening`); each value is an array of habit strings.
- `WEEK_EVENTS` — short labels shown on the week-grid; each entry pairs a `label` with a `cls` (one of the `we-*` CSS classes).
- `DAY_KEYS` / `DAY_SHORT` — Monday-first ordering used by the day buttons and week grid.

### Block `type` ↔ CSS coupling

A block's `type` is used twice: as a class on `.time-block` (`.time-block.<type>::before` paints the colored left stripe) and combined with `tag-<tag>` for the pill on the right. Valid values: `work`, `gym`, `personal`, `meeting`, `rest`, `morning`, `client`. If you add a new type, add the matching `.time-block.<type>::before`, `.tag-<type>`, and (for the week grid) `.we-<type>` rules — otherwise the block renders without color.

### Day-key gotcha

`todayKey()` (index.html:492) uses `['sun','mon','tue','wed','thu','fri','sat'][new Date().getDay()]` — Sunday-first because `Date#getDay()` returns 0 for Sunday. `DAY_KEYS` is Monday-first because that's the order the UI lists days. Don't pass a `DAY_KEYS` index where a `getDay()` index is expected, or vice versa.

## State model

Single localStorage key: **`thomas_routine_v3`**. The value is a JSON object with a flat keyspace; all reads/writes go through `loadState()` / `persist()`.

| Key pattern        | Purpose                                              |
|--------------------|------------------------------------------------------|
| `date`             | ISO date (`YYYY-MM-DD`) the rest of the state belongs to |
| `s_<day>`          | Today-tab block completion. `{ <blockIndex>: true }` |
| `cl_<clientId>`    | Per-client tasks. Keys are `am0…am3` / `pm0…pm3`.    |
| `h_<day>`          | Habit completion. Keys are `<category>_<index>`.     |
| `supp_fish` / `supp_creatine` / `supp_protein` | Supplement booleans.     |
| `gm_<ISO>` / `gn_<ISO>` / `wl_<ISO>` | Morning goals, night reflection, workout log, dated. |
| `french_notes`     | French notes. **The only field that persists across days.** |

### Daily reset semantics

`loadState()` (index.html:496) compares the stored `date` against `todayISO()`. If they differ, it constructs a fresh state object, copies over **only** keys explicitly allow-listed (currently just `french_notes`), and persists. Anything you want preserved across midnight must be added to that allow-list — otherwise it will silently disappear on the next page load on a new day.

### Bumping the schema

If you change the shape of stored state in a breaking way (renaming keys, restructuring values), bump the storage key (`thomas_routine_v3` → `_v4`) so existing users get a clean slate instead of a corrupted-looking UI. There is no migration path.

## Conventions

- Keep everything in `index.html`. Don't introduce a build tool, bundler, or split files into modules unless the user asks.
- Inline event handlers (`onclick="..."`) are the established pattern — match it rather than refactoring to `addEventListener` for new UI.
- Every state mutation must call `persist()`; renderers read from `state` directly, so forgetting `persist()` means changes vanish on reload.
- The CSS uses CSS custom properties at `:root` (index.html:11) for the palette. Add new accent colors there rather than hardcoding hex values inside rules.
