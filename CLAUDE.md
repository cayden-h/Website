# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal portfolio site for Cayden Hutcheson. Static HTML deployed on Vercel — no build step, no package manager, no tests. Edit a file, commit, push.

To preview locally, open the `.html` files directly in a browser, or run a static server from the repo root (e.g. `python3 -m http.server 8000`). The GitHub API fetch in `index.html` works either way; the `/_vercel/speed-insights/script.js` 404 locally is expected.

## Pages and their roles

- `index.html` — landing page. Hero, experience timeline (click-to-open modal), publications placeholder, projects list (fetched live from GitHub), contact.
- `project.html` — per-repo detail page. Reads `?name=<repo>` from the URL and calls the GitHub API for that repo's metadata + README. There is no router; `index.html` links here directly.
- `events.html` — hackathons / datathons / conferences directory. Card grid with click-to-open modal.
- `upcoming.html` — SWE programs + case competitions database. Searchable/filterable list with expandable rows.

## Architecture conventions

**Each HTML file is self-contained.** Tailwind config, custom CSS, all JS, and (for `events.html` / `upcoming.html`) the page's data are inlined in the same file. There are no shared CSS or JS files. Adding new content usually means editing one file end-to-end.

**Theme is duplicated per page.** The Tailwind `theme.extend.colors` block (cream/sand/sage/slate) and the Space Grotesk font setup are copy-pasted into each HTML file's `<script>tailwind.config = {...}</script>`. If you change a color or font, update every page that uses it. The palette: `cream #FAF8F5`, `sand #E8E4DE`, `sage 300/400/500/600`, custom slate scale.

**Dependencies are CDN-only:** Tailwind Play CDN, GSAP + ScrollTrigger, Google Fonts (Space Grotesk). No bundler, no `node_modules`. Do not introduce a build step without discussing first.

**Data lives in the page that renders it:**
- Experience modal content → `experienceData` object in `index.html` (keys match `data-experience` attributes on timeline items).
- Events → `events` object in `events.html` (keys match the card IDs).
- SWE programs / case competitions → `P` array in `upcoming.html`. Each entry uses terse single-letter keys (`n`, `o`, `c`, `col`, `fly`, `paid`, `div`, `self`, `fin`, `dl`, `open`, `pay`, `loc`, `desc`, `why`, `link`) — preserve the schema when adding entries; the filter/render logic depends on these exact fields.

**Projects list is dynamic.** `index.html` calls `https://api.github.com/users/cayden-h/repos` and filters out forks, the `Website` repo itself, and any repo name containing `neetcode`. To exclude another repo, add it to that filter (around `index.html:650`). The `GITHUB_USERNAME` constant is defined just above the fetch.

**`project.html` is loaded with a query string.** `?name=<repo>` is parsed in the inline script (around line 238) and used to fetch repo metadata + README from the GitHub API. Don't break that contract — `index.html` links directly to `project.html?name=<repo>`.

**Animations use GSAP + ScrollTrigger** with a consistent pattern: fade in + small Y/X translation on scroll. When adding sections, follow the existing `gsap.from(..., { scrollTrigger: { trigger, start: 'top 85%' } })` style so timing feels uniform.

## Deployment

Pushed to GitHub `main` → Vercel auto-deploys. The `/_vercel/speed-insights/script.js` reference and `window.si` shim at the bottom of each page are Vercel Speed Insights — leave them alone unless removing analytics.
