# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**NhatkyDori** ("Nhat ky Dori" = Dori's Diary) — a web-based photo diary/journal for the user's daughter. Parents can upload photos and write entries; the app displays them as a visual timeline or gallery.

## Tech Stack

- **Single-file HTML app** — all HTML, CSS, and JS in `index.html` (no build step, no bundler, no framework)
- **Vanilla JavaScript (ES2022+)** and **CSS3**
- **No package.json, no npm** — external libraries loaded via CDN only
- **GitHub Pages** hosting from `main` branch

## Deployment

```bash
git add index.html
git commit -m "description of change"
git push origin main
```

GitHub Pages serves `index.html` directly. No build or CI/CD pipeline.

- **Repo:** `https://github.com/a17255/nhatky-dori` (confirm actual repo name after creation)
- **Live URL:** `https://a17255.github.io/nhatky-dori/` (confirm after GitHub Pages is enabled)

## Architecture

Single `index.html` contains everything:
- `<style>` block: all CSS (dark theme matching container-calculator style)
- `<body>`: app markup
- `<script>` block: all application logic

## Design Conventions (from container_calculator sibling project)

- Dark GitHub-style theme: background `#0d1117`, text `#e6edf3`
- Mobile-responsive (tab layout for phones)
- No external CSS frameworks — hand-written CSS
- CDN-loaded dependencies only (e.g., Firebase if needed for backend)

## Key Constraints

- Keep everything in a single `index.html` — do not split into separate JS/CSS files
- No build tools or package managers
- Must work on mobile browsers (parents posting from phones)
