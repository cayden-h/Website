# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal portfolio site for Cayden Hutcheson. Static HTML deployed on Vercel — no build step, no package manager, no tests. Edit a file, commit, push.

To preview locally, open the `.html` files directly in a browser, or run a static server from the repo root (e.g. `python3 -m http.server 8000`). The GitHub API fetch in `index.html` works either way; the `/_vercel/speed-insights/script.js` 404 locally is expected.

## Pages and their roles

- `index.html` — landing page. Hero, experience timeline (click-to-open modal), publications placeholder, projects list (fetched live from GitHub), contact.
- `project.html` — per-repo detail page. Reads `?name=<repo>` from the URL and calls the GitHub API for that repo's metadata + README. There is no router; `index.html` links here directly.
- `events.html` — hackathons / datathons / conferences (past events), presented as a Three.js spherical gallery: the viewer sits inside a sphere tiled with event cards (the events repeat to fill it). Drag/wheel to look around, tap a card to open a full-screen detail view. A "Looking ahead" HUD pill links to `upcoming.html`.
- `upcoming.html` — searchable database of forward-looking opportunities: sophomore-only programs, hackathons, and case competitions (~120 entries). Search bar + category chips at top, click-to-expand inline detail panels.

## Architecture conventions

**Each HTML file is self-contained.** Tailwind config, custom CSS, all JS, and (for `events.html` / `upcoming.html`) the page's data are inlined in the same file. There are no shared CSS or JS files. Adding new content usually means editing one file end-to-end.

**Theme is duplicated per page.** The Tailwind `theme.extend.colors` block (cream/sand/sage/slate) and the Space Grotesk font setup are copy-pasted into each HTML file's `<script>tailwind.config = {...}</script>`. If you change a color or font, update every page that uses it. The palette: `cream #FAF8F5`, `sand #E8E4DE`, `sage 300/400/500/600`, custom slate scale.

**Dependencies are CDN-only:** Tailwind Play CDN, GSAP + ScrollTrigger, Google Fonts (Space Grotesk), and Three.js (ES module via an inline `importmap` in `events.html` only). No bundler, no `node_modules`. Do not introduce a build step without discussing first.

**Data lives in the page that renders it:**
- Experience modal content → `experienceData` object in `index.html` (keys match `data-experience` attributes on timeline items).
- Events → `events` object in `events.html`. Each entry's `photos` array points to web-optimized images in `assets/events/<key>/` (first photo = sphere card hero + detail hero; the rest fill the detail-page gallery). Raw originals live in the gitignored `Events/` folder; to add a photo run `sips -s format jpeg -s formatOptions 75 --resampleHeightWidthMax 1080 <src> --out assets/events/<key>/<name>.jpg` (sips drops EXIF rotation — check portrait shots and fix with `sips -r 90/180/270`).
- Upcoming opportunities → `P` array in `upcoming.html`. Terse-key schema: `n` (name), `o` (org · location · duration), `c` (`"swe"` for sophomore programs / `"case"` for case competitions / `"hack"` for hackathons), `col` (brand hex), boolean flags `fly` `paid` `div` `self` `fin`, plus `dl` `open` `pay` `loc` `desc` `why` `link`. The filter/render logic depends on these exact fields, and rows are bucketed into the `SECS` sections by `c` + flags. When adding entries, append at the end — source order doesn't affect grouping. Strings get HTML-escaped at render time, so write `&` (not `&amp;`) in the data.

**Projects list is dynamic.** `index.html` calls `https://api.github.com/users/cayden-h/repos` and filters out forks, the `Website` repo itself, and any repo name containing `neetcode`. To exclude another repo, add it to that filter (around `index.html:650`). The `GITHUB_USERNAME` constant is defined just above the fetch.

**`project.html` is loaded with a query string.** `?name=<repo>` is parsed in the inline script (around line 238) and used to fetch repo metadata + README from the GitHub API. Don't break that contract — `index.html` links directly to `project.html?name=<repo>`.

**Animations use GSAP + ScrollTrigger** with a consistent pattern: fade in + small Y/X translation on scroll. When adding sections, follow the existing `gsap.from(..., { scrollTrigger: { trigger, start: 'top 85%' } })` style so timing feels uniform.

**`events.html` is a fixed-viewport WebGL page, not a scroll page.** Cards are canvas-drawn textures (`drawCard`) on planes positioned by the `ROWS` lat/long table and oriented with `lookAt(0,0,0)`; the camera stays at the origin and look direction is driven by eased `lon`/`lat` (lerp in `animate()` — that lerp IS the smooth-scroll feel, don't replace it with direct assignment). Click vs drag is disambiguated by a movement threshold (`moved < 7`) in `onUp`. `openDetail`/`closeDetail` tween the tapped mesh toward the camera and fade the rest via per-mesh cloned materials — materials are cloned per card on purpose so opacity animates independently. The detail view is a DOM overlay (`#detail`), not WebGL. Up to 4 of each event's `photos` rotate across its repeated sphere cards; an event with no photos gets the sage-gradient fallback card. Because the photos are local files drawn into WebGL textures, preview `events.html` through a local server — over `file://` the canvas taints and the cards fail to render.

**`upcoming.html` toggle is DOM-mutating, not re-rendering.** `toggle(i)` finds the row by `data-idx` and animates the `.detail` panel open/closed in place; only `setCat` and `doSearch` call `render()` (which rewrites `innerHTML` and re-runs the GSAP stagger). Do not regress this — making `toggle()` re-render replays the stagger on every click and feels broken.

## Deployment

Pushed to GitHub `main` → Vercel auto-deploys. The `/_vercel/speed-insights/script.js` reference and `window.si` shim at the bottom of each page are Vercel Speed Insights — leave them alone unless removing analytics.
