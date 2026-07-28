# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal portfolio site for Cayden Hutcheson. Static HTML deployed on Vercel — no build step, no package manager, no tests. Edit a file, commit, push.

To preview locally, open the `.html` files directly in a browser, or run a static server from the repo root (e.g. `python3 -m http.server 8000`). The GitHub API fetch in `index.html` works either way; the `/_vercel/speed-insights/script.js` 404 locally is expected.

## Pages and their roles

- `index.html` — landing page. Hero, experience timeline (click-to-open modal), publications placeholder, projects list (fetched live from GitHub), contact.
- `project.html` — per-repo detail page. Reads `?name=<repo>` from the URL and calls the GitHub API for that repo's metadata + README. There is no router; `index.html` links here directly.
- `events.html` — hackathons / datathons / conferences (past events), presented as a Three.js spherical gallery: the viewer sits inside a sphere tiled with event cards (the events repeat to fill it). Drag/wheel to look around, tap a card to open a full-screen detail view. A "Looking ahead" HUD pill links to `upcoming.html`.
- `upcoming.html` — searchable database of forward-looking opportunities: sophomore-only programs, hackathons, and case competitions (~240 entries). Search bar + category chips at top, two mutually-exclusive date-sort toggles ("By open date" / "By deadline") on the right, click-to-expand inline detail panels. Every row shows a date-provenance badge (confirmed / estimated / discontinued) under its deadline.

## Architecture conventions

**Each HTML file is self-contained.** Tailwind config, custom CSS, all JS, and (for `events.html` / `upcoming.html`) the page's data are inlined in the same file. There are no shared CSS or JS files. Adding new content usually means editing one file end-to-end.

**Theme is duplicated per page.** The Tailwind `theme.extend.colors` block (cream/sand/sage/slate) and the Space Grotesk font setup are copy-pasted into each HTML file's `<script>tailwind.config = {...}</script>`. If you change a color or font, update every page that uses it. The palette: `cream #FAF8F5`, `sand #E8E4DE`, `sage 300/400/500/600`, custom slate scale.

**Dependencies are CDN-only:** Tailwind Play CDN, GSAP + ScrollTrigger, Google Fonts (Space Grotesk), and Three.js (ES module via an inline `importmap` in `events.html` only). No bundler, no `node_modules`. Do not introduce a build step without discussing first.

**Data lives in the page that renders it:**
- Experience modal content → `experienceData` object in `index.html` (keys match `data-experience` attributes on timeline items).
- Events → `events` object in `events.html`. Each entry's `photos` array points to web-optimized images in `assets/events/<key>/` (first photo = sphere card hero + detail hero; the rest fill the detail-page gallery). Raw originals live in the gitignored `Events/` folder; to add a photo run `sips -s format jpeg -s formatOptions 75 --resampleHeightWidthMax 1080 <src> --out assets/events/<key>/<name>.jpg` (sips drops EXIF rotation — check portrait shots and fix with `sips -r 90/180/270`).
- Upcoming opportunities → `P` array in `upcoming.html`. Terse-key schema: `n` (name), `o` (org · location · duration), `c` (`"swe"` for sophomore programs / `"case"` for case competitions / `"hack"` for hackathons), `col` (brand hex), boolean flags `fly` `paid` `div` `self` `fin`, plus `dl` `open` `pay` `loc` `desc` `why` `link`, and the provenance pair `v` + `vs` (see below). The filter/render logic depends on these exact fields, and rows are bucketed into the `SECS` sections by `c` + flags. When adding entries, append at the end — source order doesn't affect grouping. Strings get HTML-escaped at render time, so write `&` (not `&amp;`) in the data.

**Projects list is dynamic.** `index.html` calls `https://api.github.com/users/cayden-h/repos` and filters out forks, the `Website` repo itself, and any repo name containing `neetcode`. To exclude another repo, add it to that filter (around `index.html:693`). The `GITHUB_USERNAME` constant is defined just above the fetch (`index.html:682`).

**`project.html` is loaded with a query string.** `?name=<repo>` is parsed in the inline script (around line 238) and used to fetch repo metadata + README from the GitHub API. Don't break that contract — `index.html` links directly to `project.html?name=<repo>`.

**Animations use GSAP + ScrollTrigger** with a consistent pattern: fade in + small Y/X translation on scroll. When adding sections, follow the existing `gsap.from(..., { scrollTrigger: { trigger, start: 'top 85%' } })` style so timing feels uniform.

**`events.html` is a fixed-viewport WebGL page, not a scroll page.** Cards are canvas-drawn textures (`drawCard`) on planes positioned by the `ROWS` lat/long table and oriented with `lookAt(0,0,0)`; the camera stays at the origin and look direction is driven by eased `lon`/`lat` (lerp in `animate()` — that lerp IS the smooth-scroll feel, don't replace it with direct assignment). Click vs drag is disambiguated by a movement threshold (`moved < 7`) in `onUp`. `openDetail`/`closeDetail` tween the tapped mesh toward the camera and fade the rest via per-mesh cloned materials — materials are cloned per card on purpose so opacity animates independently. The detail view is a DOM overlay (`#detail`), not WebGL. Up to 4 of each event's `photos` rotate across its repeated sphere cards; an event with no photos gets the sage-gradient fallback card. Because the photos are local files drawn into WebGL textures, preview `events.html` through a local server — over `file://` the canvas taints and the cards fail to render.

**`upcoming.html` toggle is DOM-mutating, not re-rendering.** `toggle(i)` finds the row by `data-idx` and animates the `.detail` panel open/closed in place; only `setCat`, `doSearch`, and `setSort` call `render()` (which rewrites `innerHTML` and re-runs the GSAP stagger). Do not regress this — making `toggle()` re-render replays the stagger on every click and feels broken.

**`upcoming.html` entries carry date provenance, and it is not decorative.**
Most programs on this page have not published their next cycle's dates, so the array mixes sourced facts with inferences.
The `v` field records which is which and `verifyBadge()` renders it under the deadline: `v:1` = confirmed, a date actually read off an official or authoritative page, with `vs` holding a short source label shown in the badge (e.g. `vs:"federalreserve.gov"`); `v:0` = estimated, inferred from prior-cycle timing and not confirmed for the coming cycle; `v:2` = discontinued, covering dead programs, retired names, and eligibility mismatches (grad-only, high-school-only).
Every entry must have a `v`, and every `v:1` must have a `vs` — a confirmed badge with no source defeats the point.

**Rules for researching `upcoming.html` data.**
These exist because earlier passes got them wrong, so treat them as hard constraints rather than style preferences:
- **A missing job posting is not evidence a program is dead.** These programs are off-cycle for most of the year, so checking whether a listing is live right now is misleading. Research when the program opened in previous cycles and store that as an explicit `v:0` estimate instead.
- **`v:2` requires positive evidence** — an explicit statement from the organisation, a domain that fails DNS, or an eligibility rule that excludes undergrads. Never infer it from silence.
- **Only compensated opportunities belong here.** Exclude unpaid programs and anything the student pays for. Exceptions: summits and insight events that compensate via flights, accommodation or goods; unpaid bank insight programs (they are the documented early-ID pipeline); and hackathons (free entry, food, swag and prize pools count).
- **Verify agent research before committing it.** Subagent reports have been wrong in both directions on this page — fabricated dates presented as sourced, and confident retractions of claims that were actually correct. Check any high-impact claim against the page yourself.
- **Program names are a common failure mode.** Whole clusters of plausible-sounding entries ("Sophomore Insight Day" at various banks) turned out not to exist. If a firm's own student page enumerates its programs and the name is absent, that is positive evidence the name is wrong — unlike a missing posting.

**`upcoming.html` date sorting reuses one parser across two views.** `setSort(mode, btn)` toggles `sortMode` between `''` (default category grouping via `SECS`), `'open'`, and `'dl'`; the two modes are mutually exclusive and clicking the active one returns to category view. When a sort mode is on, `render()` buckets the filtered list by month and orders soonest→furthest, using `dateKey()` to parse the freeform `open`/`dl` strings (e.g. `~Oct 2026`, `~Jan–Feb 2027`, `~Spring 2027`, `Aug 2026 / Mar 2027`) into a sortable `year*12 + month`. Rules: seasons map to a representative month (spring→Mar, summer→Jun, fall→Sep, winter→Dec); multi-date entries take the earliest; a month with no year resolves to its next upcoming occurrence relative to today; anything with no parseable month lands in a "Rolling / ongoing" bucket pinned last. Sort works alongside the active category/search filter.
Status strings deliberately fall into that last bucket too (`Rolling / closes when filled`, `Program discontinued`, `Not open to college students`), which is why they are safe to use in `dl`/`open`.
Beware the inverse: any month name anywhere in the string is parsed, so a status string that happens to contain one (`"closed since March"`) will sort into that month.
Two shapes worth copying — `OPEN NOW (~July 2026)` surfaces live opportunities while still sorting correctly, and a hard date is written plainly as `Sept 30, 2026 (5pm ET)`.

## Deployment

Pushed to GitHub `main` → Vercel auto-deploys. The `/_vercel/speed-insights/script.js` reference and `window.si` shim at the bottom of each page are Vercel Speed Insights — leave them alone unless removing analytics.
