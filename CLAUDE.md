# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **static single-page blog CMS** for "Ms Ngoc Academy" — a Vietnamese English-language education platform. The entire application lives in one file: `index.html`. There is no build system, no package manager, and no backend framework.

## Architecture

### Single-file SPA
Everything — HTML structure, CSS styles (~600 lines), and JavaScript logic (~200 lines) — is embedded in `index.html`. External dependencies are loaded via CDN:
- **Marked.js** (`cdn.jsdelivr.net`) — renders Markdown content from the API
- **Google Fonts** — Be Vietnam Pro (headings) + Inter (body)
- **Facebook Comments SDK** — discussion section on detail view

### Data Layer (Google Apps Script)
Content is fetched at runtime from a Google Sheets backend via a Google Apps Script web app:

```javascript
const API_URL = "https://script.google.com/macros/s/.../exec";
```

The API returns JSON with this schema:
```json
{
  "data": [
    {
      "Title": "string",
      "Description": "string",
      "Category": "string",
      "Content": "markdown string",
      "ImageURL": "string",
      "Date": "YYYY-MM-DD"
    }
  ]
}
```

The `Content` field is raw Markdown, rendered client-side by Marked.js with `breaks: true, gfm: true`.

### Client-Side Routing
Navigation uses URL query parameters — no hash routing, no client-side router:
- **Home/grid view:** `index.html` (no params)
- **Detail view:** `index.html?id=<index>` — `id` maps to the array index (`originalId`) assigned after fetching

### JavaScript Application State
```javascript
let allPosts = [];        // master list from API
let filteredPosts = [];   // search/category-filtered subset
let currentPage = 1;
const POSTS_PER_PAGE = 6;
```
Key functions: `initApp()` → `setupCategories()` + `renderGrid()` | `filterPosts()` | `renderDetail(post)` | `changePage(dir)`

## CSS Design System

CSS custom properties define the entire color palette:

| Variable | Value | Usage |
|---|---|---|
| `--xanh-duong-dam` | `#0A2540` | Primary dark navy |
| `--vang-anh-kim` | `#D4AF37` | Gold accent, hover states |
| `--do-do` | `#8A1C2C` | Burgundy, category badges |
| `--bg-body` | `#F8FAFE` | Page background |
| `--text-muted` | `#5A6B7F` | Secondary text |

All interactive hover effects transition to gold (`--vang-anh-kim`). Card animations use `fadeInUp` keyframes with staggered `animation-delay` per card index.

## Content File Format

Lesson content files in the repo (like `Động từ To Be (am/is/are)`) are tab-separated data files intended for import into Google Sheets. The column order is:

```
Title \t Date \t ImageURL \t "Content (Markdown)" \t Description \t Category
```

## Development

No build step. To develop locally, serve `index.html` with any static HTTP server:

```bash
python3 -m http.server 8000
# or
npx serve .
```

Open `http://localhost:8000` — the page fetches live data from the Google Apps Script API. There is no mock/local data mode.

To add new lesson content, add a row to the connected Google Sheet. The frontend auto-discovers all categories from the data.

## Responsive Breakpoint

Single breakpoint at `768px`. Below this width: header padding reduces, filter bar stacks vertically, article view padding collapses.
