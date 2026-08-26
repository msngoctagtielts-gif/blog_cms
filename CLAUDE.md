# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A **static single-page blog CMS** for "Ms Ngoc Academy" / "Ms. Ngoc Elite English" — a Vietnamese English-language education platform. The published site is a lesson library: a searchable/filterable card grid plus a Markdown article detail view with Facebook comments.

The entire application is one file: `index.html` (808 lines). There is **no build system, no package manager, no test suite, no backend framework, and no CI**. The repository root contains only:

| Path | What it is |
|---|---|
| `index.html` | The whole application (HTML + CSS + JS) |
| `CLAUDE.md` | This file |
| `Speak Now 1 - Unit 1` | Lesson content row (TSV, extensionless) |
| `Speak Now 1 - Unit 2` | Lesson content row (TSV, extensionless) |
| `Speak Now 1 - Unit 4` | Lesson content row (TSV, extensionless) |
| `Động từ To Be (am/is/are)` | Lesson content row — **see the slash gotcha below** |

Content files are **staging data for manual copy-paste into Google Sheets**, not something the site reads. The site never loads any file from this repo except `index.html`.

## Architecture

### Single-file SPA — `index.html` map

| Lines | Contents |
|---|---|
| 1–15 | `<head>`, Google Fonts preconnect + link, Marked.js CDN `<script>` |
| 16–613 | `<style>` — the entire stylesheet; `:root` custom properties at 17–31 |
| 614–615 | `#fb-root` + Facebook SDK script (**inside `<head>`** — see gotchas) |
| 619–633 | `<header class="modern-header">` — avatar, gradient title, badges |
| 635–650 | `.container` — filter bar, `#app-container`, `#pagination-container` |
| 652–806 | Application `<script>` |

CDN dependencies (no local vendoring, no SRI hashes):
- **Marked.js** (`cdn.jsdelivr.net/npm/marked/marked.min.js`, unpinned `latest`) — renders the Markdown `Content` field
- **Google Fonts** — Be Vietnam Pro (headings) + Inter (body)
- **Facebook Comments SDK** (`connect.facebook.net/vi_VN/sdk.js`, v19.0) — discussion section on the detail view

### Data layer (Google Apps Script → Google Sheets)

All content is fetched at runtime; nothing is bundled:

```javascript
// index.html:654
const API_URL = "https://script.google.com/macros/s/AKfycbx.../exec";
```

`initApp()` appends a cache-buster (`?timestamp=` + `Date.now()`) to every request. Response schema:

```json
{ "data": [ { "Title": "…", "Date": "…", "ImageURL": "…",
              "Content": "markdown", "Description": "…", "Category": "…" } ] }
```

Field names are **capitalized and must match the Google Sheet header row exactly** — the JS reads `post.Title`, `post.Category`, etc. with no normalization. Renaming a sheet column silently breaks the field everywhere.

`Content` is raw Markdown, parsed client-side with `marked.setOptions({ breaks: true, gfm: true })` (index.html:656). GFM tables are used heavily by the lesson content.

### Client-side routing

URL query parameters only — no hash routing, no router library:

- **Grid view:** `index.html` (no params) → `setupCategories()` + `renderGrid()`
- **Detail view:** `index.html?id=<n>` → `renderDetail(allPosts[n])`

`id` is the **array index** (`originalId`), assigned in `initApp()` after the fetch:

```javascript
allPosts = json.data.map((post, idx) => { post.originalId = idx; return post; });
```

**Permalinks are therefore positional, not stable.** Inserting or reordering a row in the Google Sheet shifts every subsequent `?id=`. Because the Facebook comments widget is keyed on `data-href="${window.location.href}"`, a shift also reattaches existing comment threads to the wrong articles. Append new rows to the end of the sheet; do not insert or reorder.

An `?id=` that doesn't resolve (`allPosts[postId]` falsy) falls through to the grid view rather than showing a 404.

### Application state and control flow

```javascript
let allPosts = [];          // master list from the API, each tagged with originalId
let filteredPosts = [];     // search + category subset
const POSTS_PER_PAGE = 6;
let currentPage = 1;
```

```
initApp()  ─┬─ ?id present ──→ renderDetail(post) ──→ FB.XFBML.parse()
            └─ otherwise ────→ setupCategories() + renderGrid()

filterPosts()  → resets currentPage = 1 → renderGrid()
changePage(±1) → renderGrid() → smooth-scroll to #app-container
```

| Function | Line | Notes |
|---|---|---|
| `formatDate(s)` | 663 | `new Date(s)` → `DD/MM/YYYY`; returns the raw string when unparseable, `'Mới đăng'` when empty |
| `initApp()` | 673 | fetch → map ids → route; catch renders a Vietnamese connection-error panel |
| `setupCategories()` | 699 | builds the `<select>` from a `Set` of non-empty `Category` values — categories are auto-discovered, never hardcoded |
| `filterPosts()` | 708 | matches lowercase keyword against `Title` + `Description` only (**not** `Content`), ANDed with the category |
| `renderGrid()` | 722 | slices `filteredPosts`, builds cards, shows pagination only when `totalPages > 1` |
| `changePage(dir)` | 769 | no clamping — relies on the buttons being `disabled` at the bounds |
| `renderDetail(post)` | 775 | article markup + FB comments; re-parses XFBML since the widget is injected after SDK load |

All rendering is template-literal string concatenation into `innerHTML`; there is no framework, no vDOM, and no escaping. Field values from the sheet are interpolated raw and `marked.parse()` runs without a sanitizer — the sheet is treated as trusted authored content. Keep it that way: do not wire this UI to user-submitted data without adding escaping/sanitization.

Fallbacks worth knowing: missing `ImageURL` falls back to a hot-linked Unsplash photo (index.html:735); missing `Description` falls back to the first 110 characters of `Content`; missing `Title`/`Category` fall back to Vietnamese placeholder strings.

## CSS design system

All colors live in `:root` (index.html:17–31). Use the variables — do not introduce new literal hex values.

| Variable | Value | Usage |
|---|---|---|
| `--xanh-duong-dam` | `#0A2540` | Primary dark navy — titles, headings |
| `--xanh-duong-sang` | `#1A4A7A` | Lighter navy for gradients |
| `--vang-anh-kim` | `#D4AF37` | Gold accent — **every** hover/focus state |
| `--do-do` | `#8A1C2C` | Burgundy — category badges, links, error text |
| `--do-do-nhat` | `#9E3A47` | Soft burgundy for effects |
| `--bg-body` | `#F8FAFE` | Page background |
| `--bg-card` | `#FFFFFF` | Card surface |
| `--text-main` | `#1F2A3E` | Body text |
| `--text-muted` | `#5A6B7F` | Secondary text, dates |
| `--border-color` | `#E9EEF4` | Hairlines |
| `--glass-bg` | `rgba(255,255,255,0.92)` | Frosted filter bar |
| `--shadow-sm` / `--shadow-hover` | navy-tinted | Resting / hover elevation |

Conventions:
- **Gold on hover, always.** Cards, buttons, links, and focus rings all transition to `--vang-anh-kim`.
- **Generous radii.** Cards `28px`, article view `36px`, pills/buttons `40–60px`.
- **Entrance animation.** `.post-card` uses the `fadeInUp` keyframes with a staggered `animation-delay: ${idx * 0.08}s` applied inline per card in `renderGrid()`.
- **Typography.** `Be Vietnam Pro` for headings and card/article titles, `Inter` for body. (`.art-content code` declares `'Be Vietnam Pro', monospace` — a quirk, not a real mono stack.)
- **Grid.** `repeat(auto-fill, minmax(340px, 1fr))`, `36px` gap.
- Comments in the stylesheet are in Vietnamese; match that when editing CSS.

### Responsive

A single breakpoint at `768px` (index.html:607–612). Below it: header padding shrinks, the filter bar stacks vertically and unsticks (`top: 0`), the article view padding collapses, grid gap tightens. There is no tablet/desktop tier beyond this.

## Content file format

Lesson files in the repo are **one-row TSV records** staged for pasting into the Google Sheet. Six tab-separated columns, in this order:

```
Title <TAB> Date <TAB> ImageURL <TAB> "Content (Markdown)" <TAB> Description <TAB> Category
```

Rules the existing files follow:
- The `Content` column is wrapped in double quotes and spans many physical lines (CSV-style quoted multiline field). Every file therefore has exactly **5 tab characters**: 3 on the first line (`Title → Date → ImageURL → "Content…`) and 2 on the last line, after the closing `"` (`…"  → Description → Category`).
- Files are extensionless and named after the lesson (e.g. `Speak Now 1 - Unit 2`).
- Content is bilingual: Vietnamese explanation around English target language, with `| English | Vietnamese |` GFM tables, `##` section headings, `>` blockquote summaries, and emoji.
- Typical section order: Mục tiêu (goals) → Vocabulary → Grammar → Mẫu câu/hội thoại → Luyện tập + Đáp án → Tổng kết.
- `Category` values seen so far: `Speak Now 1`, `Ngữ Pháp`. `ImageURL` is a shared postimg.cc logo.

### Date format gotcha

The existing files use `D-M-YYYY` (e.g. `16-6-2026`, `2-5-2026`), but `formatDate()` calls bare `new Date(string)`, which parses **M-D-YYYY**. The results are inconsistent:

| Sheet value | `new Date()` result | What the site shows |
|---|---|---|
| `16-6-2026` | Invalid Date | `16-6-2026` (raw, unformatted) |
| `2-5-2026` | Feb 5 2026 | `05/02/2026` — **wrong month/day** |
| `2026-06-16` | Jun 16 2026 | `16/06/2026` ✅ |

Use **ISO `YYYY-MM-DD`** for any new content. Changing existing rows to ISO is a safe, self-contained fix if asked.

### Filename slash gotcha

`Động từ To Be (am/is/are)` is *intended* as a single filename, but the `/` characters make it a real directory chain on disk and in git:

```
Động từ To Be (am/          ← directory
└── is/                     ← directory
    └── are)                ← the actual file
```

Always quote the full path — `cat "Động từ To Be (am/is/are)"` works; `ls` on the root shows only `Động từ To Be (am`. Do not "fix" this by renaming unless asked; it would change tracked paths in git. New content files should avoid `/` in their names.

## Development

No build step, no install, no lint, no tests. Serve the directory with any static server:

```bash
python3 -m http.server 8000
# or
npx serve .
```

Open `http://localhost:8000`. The page always fetches live data from the Google Apps Script endpoint — **there is no mock or offline data mode**, so the grid will be empty (or show the error panel) without network access to `script.google.com`. To work on layout offline, temporarily stub `allPosts` at the top of `initApp()`.

Because everything is one file, "deploying" is committing `index.html`; the site is served as a static page (GitHub Pages / any static host).

### Adding lesson content

1. Write the lesson as a one-row TSV file in the repo root, following the six-column format above (this gives the content review history in git).
2. **Append** the row to the connected Google Sheet — never insert above existing rows (it shifts every `?id=` permalink).
3. The frontend picks it up on the next load; categories are auto-discovered from the data, so a new `Category` needs no code change.

### Editing `index.html`

- Keep the single-file structure. Do not split out `.css`/`.js`, add a bundler, or introduce a framework unless explicitly asked.
- Reference colors through the `:root` variables.
- User-facing strings are **Vietnamese** — match the existing tone (warm, slightly formal, emoji-friendly: "Đang tải kho tàng bài giảng…", "Khám phá ngay →").
- Test by hand in a browser: grid render, search, category filter, pagination, detail view, back link, and mobile width. There is nothing automated to run.

## Known issues / gotchas

- **FB SDK placement.** `#fb-root` and the SDK `<script>` sit inside `<head>` (index.html:614–615), after `</style>`. Browsers hoist them into `<body>`, so it works, but it is invalid HTML — leave it alone or fix it deliberately, not incidentally.
- **Avatar hot-link.** The header avatar (index.html:622) is a Facebook CDN URL with an expiry token (`oe=…`). It will eventually 404; replace with a stable host (e.g. postimg.cc, like the lesson images) when it breaks.
- **Unpinned Marked.js.** The CDN URL tracks `latest`, so an upstream major release can change rendering without any commit here.
- **Search ignores `Content`.** Only `Title` and `Description` are searched — a term that appears solely in the lesson body will not match.
- **`id` is positional.** See routing above; this is the single most damaging thing to get wrong when editing the sheet.
- **No escaping anywhere.** Sheet fields go straight into `innerHTML` and Markdown is parsed unsanitized.
- **Detail view has no pagination or "next lesson" link** — the only exit is the back link to the library.

## Git conventions

Short imperative commit subjects, no body, no scope prefixes — matching the existing history:

```
Add Speak Now 1 Unit 4 lesson: Let's Eat!
Add CLAUDE.md and Speak Now 1 Unit 1 lesson content
Update index.html
```

Work happens on `claude/*` branches merged into `main` via pull request.
